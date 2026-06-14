# Karmik — Hugging Face Integration

## Overview

Karmik's model catalog currently covers a curated set of GGUF models (Phi, Gemma, Llama,
Mistral, etc.). This spec adds a direct Hugging Face Hub integration that lets users:

1. **Browse** Hugging Face models from within Karmik
2. **Download** GGUF quantizations directly (no conversion needed)
3. **Convert** non-GGUF HuggingFace models to GGUF on-device using `llama.cpp`'s
   `convert_hf_to_gguf.py` (desktop only — Python required)
4. **Use Hugging Face Inference API** as a remote provider (serverless or dedicated endpoints)
5. **Cache models locally** with the same download manager used for curated models

---

## Hugging Face Hub Browser

A new **"From HuggingFace"** tab in the Model Manager (Settings → Models → Browse):

```
Model Library
═════════════════════════════════════════════
  Curated   |  Hugging Face   |  Installed
═════════════════════════════════════════════

🔍 [Search HuggingFace models...          ]

Filters: [Task ▾]  [Size ▾]  [Format ▾]  [License ▾]

GGUF Models (ready to use)
─────────────────────────────────────────────
  📦 Qwen2.5-7B-Instruct-GGUF
     by Qwen                    ★ 4.8 · 12.4K downloads
     Q4_K_M: 4.7GB  Q8_0: 8.1GB
     [Download Q4_K_M]  [Download Q8_0]

  📦 Mistral-7B-Instruct-v0.3-GGUF
     by MistralAI               ★ 4.7 · 9.1K downloads
     Q4_K_M: 4.1GB  Q8_0: 7.2GB
     [Download Q4_K_M]  [Download Q8_0]

  📦 DeepSeek-R1-Distill-Qwen-7B-GGUF
     by lmstudio-community      ★ 4.9 · 5.2K downloads
     Q4_K_M: 4.4GB
     [Download Q4_K_M]

Non-GGUF Models (conversion required)
─────────────────────────────────────────────
  🔄 Qwen2.5-7B-Instruct (safetensors)
     by Qwen                    ~15GB download → ~4.7GB GGUF
     [Download & Convert]  ← desktop only; requires Python 3.10+
─────────────────────────────────────────────
```

### Search API

The HuggingFace Hub browser uses the **Hugging Face Hub API** (public, no auth required
for public models):

```
GET https://huggingface.co/api/models
  ?search={query}
  &filter=gguf          ← for the GGUF tab
  &sort=downloads
  &limit=20
```

**Auth**: optional HuggingFace access token (stored in platform keystore) for:
- Access to gated models (Llama 3, Mistral v0.2, etc. — require accepting terms)
- Accessing private models in the user's own HF account

Token is entered in Settings → Models → HuggingFace → "Access token".

**Privacy Mode**: Hub API calls are internet requests — blocked in Privacy Mode.
Downloaded models (already cached) can still be used in Privacy Mode.

---

## GGUF Download Flow

When a user taps **[Download Q4_K_M]** on a GGUF model:

1. Karmik calls `GET https://huggingface.co/api/models/{owner}/{repo}` to get file metadata
2. Finds the GGUF file matching the selected quantization (e.g., `*Q4_K_M*.gguf`)
3. Enqueues it in the existing **DownloadManager** (from spec 03-model-runtime.md):
   - Resume-capable (uses HTTP Range headers)
   - Progress shown in the download list
   - SHA-256 hash verified on completion
4. On completion: model appears in the Installed tab with full metadata

**The downloaded GGUF model is usable identically to curated models** — it goes through the
same `ModelCatalogEntry` + `LlamaCppRuntime` path. No special handling needed.

**Metadata auto-detection**: Karmik reads the model's HuggingFace card to extract:
- `context_length` (from `config.json` or README)
- `chat_template` (from `tokenizer_config.json`)
- Vision capability (from model architecture in `config.json`)
- Recommended system prompt (from model card)

These are stored in `karmik_models.db` alongside the model entry.

---

## Non-GGUF Conversion (Desktop Only)

For models that only have `safetensors` or `pytorch_model.bin` files (not pre-quantized
to GGUF), Karmik can convert them locally using `llama.cpp`'s conversion script.

**Requirements**:
- macOS, Windows, or Linux (not Android/iOS — too slow/memory-constrained)
- Python 3.10+ installed
- ~3× model size in free disk space (original + conversion intermediate + GGUF output)
- ~15–60 minutes for a 7B model (CPU-bound)

**Flow**:

```
User taps [Download & Convert]
  │
  ▼
Karmik checks Python is available (shell.run "python3 --version")
  │ if not → shows "Python 3.10+ required" with download link
  ▼
Karmik downloads llama.cpp's convert_hf_to_gguf.py (or uses bundled copy)
  │
  ▼
Karmik downloads the safetensors files from HuggingFace (streamed, resumable)
  │
  ▼
Karmik runs: python3 convert_hf_to_gguf.py {model_dir} --outtype q4_k_m
             (in a subprocess, with progress piped to a log view)
  │
  ▼
Conversion complete → GGUF file added to model catalog → original files deleted
```

**Progress UI** (a new sheet in the Model Manager):

```
Converting Qwen2.5-7B-Instruct
──────────────────────────────────────────────
  ✓ Downloaded 14.8GB safetensors
  ⟳ Converting to GGUF Q4_K_M...  (12 / 32 layers — ~18 minutes remaining)

  [Cancel]
──────────────────────────────────────────────
  Log output:
  Loading model metadata...
  Processing layer 0/32: attention.q_proj
  Processing layer 1/32: attention.k_proj
  ...
```

**Cancellation**: if cancelled mid-conversion, partial files are deleted (no orphans).

**Error handling**: if conversion fails, the error output from `convert_hf_to_gguf.py` is
shown in the log view. Common failures: insufficient disk space, Python dependency missing
(`transformers`, `sentencepiece` packages — Karmik shows the missing `pip install` command).

---

## Hugging Face Inference API (Remote Provider)

In addition to local models, Karmik supports the **Hugging Face Inference API** as a
remote provider — useful for models too large to run locally (70B+).

Two API types:

### Serverless Inference API

HuggingFace's free/pay-per-use hosted inference for thousands of public models.

```dart
class HuggingFaceServerlessAdapter extends ProviderAdapter {
  final String apiKey;       // HF access token
  final String modelId;      // "meta-llama/Llama-3.1-70B-Instruct"

  // Endpoint: https://api-inference.huggingface.co/models/{modelId}
  // Format: OpenAI-compatible chat completions (for Inference Endpoints)
  //         or HF native API (for non-chat models)
}
```

**Cost**: metered by HF (free tier: ~1000 requests/month per model; paid: per-token).

### Dedicated Inference Endpoints

For users with their own HuggingFace Inference Endpoints (custom-deployed model on HF
infrastructure, billed per hour):

```dart
class HuggingFaceEndpointAdapter extends ProviderAdapter {
  final String endpointUrl;  // "https://xyz.endpoints.huggingface.cloud"
  final String apiKey;       // HF access token
  // OpenAI-compatible API — treated identically to other OpenAI-compat providers
}
```

Both are configured in Settings → Providers → "Add provider" → "Hugging Face".

**Privacy Mode**: HF Inference API is internet — blocked in Privacy Mode.

---

## Model Card Display

When a user views a HuggingFace model in the browser, Karmik fetches and displays the
model card (README.md) from the Hub:

```
Mistral-7B-Instruct-v0.3-GGUF
──────────────────────────────────────────────
Author: MistralAI  ·  License: Apache 2.0
Context: 32K tokens  ·  Architecture: Mistral

Downloads available:
  mistral-7b-instruct-v0.3.Q4_K_M.gguf  · 4.1GB   [Download]
  mistral-7b-instruct-v0.3.Q8_0.gguf    · 7.2GB   [Download]
  mistral-7b-instruct-v0.3.IQ2_XXS.gguf · 2.2GB   [Download]

Model Card:
  Mistral 7B Instruct v0.3 is an instruct fine-tuned version...
  [Read more ▾]

HuggingFace page ↗
```

---

## User's Own HuggingFace Models

Users can upload their own fine-tuned models (or private GGUF models) to their HuggingFace
account and use the same HuggingFace browser in Karmik to download them:

- Requires HuggingFace access token with read access to the private repo
- Private repos shown with a 🔒 lock icon
- Download flow identical to public models

**Note**: fine-tuning (creating these models) is outside Karmik's scope (spec 01 Non-Goal:
"A model training or fine-tuning platform"). Karmik is the consumer, not the creator.

---

## Local Model Gallery (Installed Tab)

The Installed tab in Model Manager shows all locally installed models — both curated and
HuggingFace-sourced — with metadata:

```
Installed Models
─────────────────────────────────────────────
  Phi-3.8B Q4_K_M                [Set active ✓]  [Delete]
  Source: Karmik curated  ·  2.2GB  ·  Context: 128K

  Mistral-7B-Instruct Q4_K_M     [Set active]     [Delete]
  Source: HuggingFace  ·  4.1GB  ·  Context: 32K

  Qwen2.5-7B-Instruct Q4_K_M    [Set active]     [Delete]
  Source: HuggingFace  ·  4.7GB  ·  Context: 128K
─────────────────────────────────────────────
  Total: 10.8GB used  (Free: 45.2GB)
```

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| HF Hub browser | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| GGUF download | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Non-GGUF conversion | — | — | ✓ | ✓ | ✓ | — |
| HF Inference API (remote) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Private model access | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Gated model access (token) | ✓ | ✓ | ✓ | ✓ | ✓ | — |

**Android/iOS conversion**: on-device conversion of large safetensors models is impractical
(memory, storage, time). Android and iOS users should download pre-quantized GGUF files.
The "Download & Convert" option is hidden on mobile with a tooltip: "Conversion requires
a desktop device."

**Web**: model download and local inference not available on Web. HuggingFace Inference API
works on Web (remote calls to HF's servers).
