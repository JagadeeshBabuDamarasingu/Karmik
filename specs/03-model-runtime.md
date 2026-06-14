# Karmik — Model Runtime

## Overview

The Model Runtime layer provides a single abstracted interface for inference, decoupling the
orchestration engine from any specific backend. The same agent code runs against a local GGUF
model, an ONNX model, or a cloud API without changes.

## ModelRuntime Interface

```dart
abstract class ModelRuntime {
  /// Human-readable name shown in UI ("llama.cpp", "Gemini Flash", etc.)
  String get name;

  /// Whether this runtime requires network access
  bool get isRemote;

  /// Load a model into memory. Throws if device capability check fails.
  Future<void> load(ModelConfig config);

  /// Unload model and free memory
  Future<void> unload();

  /// Whether a model is currently loaded and ready
  bool get isLoaded;

  /// Stream tokens as they are generated
  Stream<String> infer(InferenceRequest request);

  /// Cancel an in-progress inference
  Future<void> cancel();

  /// Introspect what this runtime supports
  RuntimeCapabilities get capabilities;
}

class InferenceRequest {
  final List<Message> messages;
  final List<ToolDefinition> tools;    // empty if no tools needed
  final InferenceParams params;        // temperature, top_p, max_tokens, etc.
}

class RuntimeCapabilities {
  final bool supportsToolUse;
  final bool supportsVision;           // can process image inputs
  final bool supportsStreaming;
  final int maxContextTokens;
  final List<String> supportedFormats; // ["gguf", "onnx", "mlc"]
}
```

## Supported Backends

### 1. llama.cpp (Primary Local Backend)

- **Format**: GGUF
- **Integration**: Dart FFI → llama.cpp C API (compiled as `.so` for Android ARM64)
- **Quantization support**: Q4_K_M, Q5_K_M, Q8_0, F16
- **Tool use**: JSON grammar-constrained decoding for reliable tool call parsing
- **Context window**: up to model's native limit (typically 4K–128K depending on model)
- **Models**: Gemma 4 2B/4B, Phi-4-mini, Qwen 2.5 3B/7B, DeepSeek-R1-Distill-Qwen-1.5B, Llama 3.2 3B

### 2. MLC-LLM (Secondary Local Backend)

- **Format**: MLC-compiled models (custom binary format)
- **Integration**: Android AAR via Flutter plugin
- **Advantage**: Better NPU/GPU utilization on supported Snapdragon/MediaTek chips
- **Tradeoff**: Smaller model catalog, requires pre-compilation

### 3. ONNX Runtime (Tertiary Local Backend)

- **Format**: ONNX
- **Integration**: ONNX Runtime Android library via platform channel
- **Use case**: Embedding models (for vector memory), lightweight classification tasks, vision models
- **Not used for primary chat inference** — llama.cpp or MLC-LLM preferred for that

### 4. Cloud Adapters

Implemented as `ModelRuntime` subclasses. See `specs/04-provider-adapters.md`.

## Model Configuration

```dart
class ModelConfig {
  final String id;           // unique identifier, e.g. "gemma-4-2b-q4km"
  final String displayName;  // "Gemma 4 2B (Q4_K_M)"
  final String backendType;  // "llamacpp" | "mlc" | "onnx" | "openai_compat" | "gemini" | "anthropic"
  final String? localPath;   // path to weights on device (null for remote)
  final ModelMetadata metadata;
  final HardwareRequirements requirements;
}

class ModelMetadata {
  final int parametersBillions;     // e.g. 2 for "2B"
  final String architecture;        // "gemma", "llama", "phi", "qwen"
  final int contextWindowTokens;
  final bool supportsToolUse;
  final bool supportsVision;
  final String license;
  final String sourceUrl;           // HuggingFace repo URL
}

class HardwareRequirements {
  final int minRamMb;          // minimum device RAM required
  final int diskSpaceMb;       // compressed download size
  final int? minAndroidApi;    // null = any
  final bool requiresNpu;
}
```

## Model Catalog

The catalog is a JSON file bundled with the app and periodically refreshed from a CDN.

```json
{
  "models": [
    {
      "id": "gemma-4-2b-q4km",
      "displayName": "Gemma 4 2B (Q4_K_M)",
      "backendType": "llamacpp",
      "downloadUrl": "https://huggingface.co/...",
      "metadata": {
        "parametersBillions": 2,
        "architecture": "gemma",
        "contextWindowTokens": 8192,
        "supportsToolUse": true,
        "supportsVision": false
      },
      "requirements": {
        "minRamMb": 4096,
        "diskSpaceMb": 1500,
        "minAndroidApi": 26
      }
    }
  ]
}
```

**Recommended default catalog (launch)**:

| Model | Params | VRAM approx | Tool Use | Vision | Tier |
|---|---|---|---|---|---|
| Gemma 4 2B Q4_K_M | 2B | ~1.5GB | Yes | No | Entry (4GB RAM) |
| Gemma 4 4B Q4_K_M | 4B | ~2.7GB | Yes | Yes | Mid (6GB RAM) |
| Phi-4-mini Q4_K_M | 3.8B | ~2.5GB | Yes | No | Mid (6GB RAM) |
| Qwen 2.5 7B Q4_K_M | 7B | ~4.5GB | Yes | No | High (8GB RAM) |
| DeepSeek-R1 1.5B Q8 | 1.5B | ~1.7GB | No | No | Entry (reasoning) |

## Device Capability Check

Run before allowing a model download. Checks are gating — download button is hidden/disabled,
not just warned, if the device fails.

```dart
class DeviceCapabilityChecker {
  Future<CapabilityCheckResult> check(HardwareRequirements requirements) async {
    final totalRamMb = await _getTotalRam();
    final availableDiskMb = await _getAvailableDisk();
    final androidApi = await _getAndroidApi();

    return CapabilityCheckResult(
      ramOk: totalRamMb >= requirements.minRamMb,
      diskOk: availableDiskMb >= requirements.diskSpaceMb,
      apiOk: androidApi >= (requirements.minAndroidApi ?? 21),
      // Conservative: use 80% of total RAM as usable threshold
      // (OS + other apps consume the rest)
      totalRamMb: totalRamMb,
      usableRamMb: (totalRamMb * 0.8).floor(),
    );
  }
}
```

## Model Download Manager

- Resumable chunked downloads (using `dio` with range requests)
- SHA-256 checksum verification before marking download complete
- Downloads stored in app-private storage (`getApplicationDocumentsDirectory()`)
- Progress reported as a `Stream<DownloadProgress>` (bytes downloaded, total, speed, ETA)
- Multiple concurrent downloads not allowed (queue-based)
- Automatic retry on network interruption (up to 3 attempts)

### Download State Machine

```
         user enqueues
idle ──────────────────► queued
                            │
                    slot opens (prev download done/failed)
                            │
                            ▼
                       downloading ◄──── resume (user or auto-retry)
                         │     │
              user pause │     │ network drop / app kill
                         │     │
                         ▼     ▼
                        paused (partial file retained on disk)
                            │
                    user resumes or app restarts
                            │
                            ▼
                       downloading
                            │
                   all bytes received
                            │
                            ▼
                        verifying (SHA-256 check)
                         │     │
              checksum ok │     │ checksum fail
                         │     │
                         ▼     ▼
                      complete  failed
                                │
                       partial file deleted, re-queued (up to 3 auto-retries),
                       then user-visible error if retries exhausted
```

**State persistence**: Download state and bytes-received offset are written to `karmik_models.db`
after each chunk so that a process kill mid-download resumes from the last written position on
next launch. The partial `.gguf.part` file is kept alongside the target path until verification
succeeds, then renamed to the final filename atomically.

## Per-Chat Model Selection and Hot-Swap

- Each chat/agent session can specify a `modelOverride` in its config
- If no override: use the agent's default model
- If no agent default: use the global default model (user setting)
- Hot-swap: when a session switches models mid-conversation, the current model is unloaded and
  the new one loaded. Context is preserved in the message history (re-sent with next inference).
- Only one model loaded in memory at a time (memory constraint). KV-cache is cleared on swap.

## Runtime Selection Logic

```
User requests inference
        │
        ├─ Model config has backendType: "llamacpp" → LlamaCppRuntime
        ├─ Model config has backendType: "mlc"      → MlcRuntime
        ├─ Model config has backendType: "onnx"     → OnnxRuntime
        ├─ Model config has backendType: "openai_compat" → OpenAICompatRuntime
        ├─ Model config has backendType: "gemini"   → GeminiRuntime
        └─ Model config has backendType: "anthropic" → AnthropicRuntime
```

The `ModelRuntimeFactory` resolves this mapping and returns the appropriate `ModelRuntime` instance.
