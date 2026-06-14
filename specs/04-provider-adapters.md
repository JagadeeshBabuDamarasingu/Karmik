# Karmik — Remote Provider Adapters

## Overview

Remote providers are implemented as `ModelRuntime` subclasses. There are three adapters:

1. **OpenAICompatAdapter** — covers 80%+ of providers (one adapter, many providers)
2. **GeminiAdapter** — dedicated for Google Gemini (proprietary API, best Android story)
3. **AnthropicAdapter** — dedicated for Claude (proprietary Messages API, most-requested by name)

Remote inference is **opt-in per session or per agent**. All credentials stored in Android Keystore.

## Adapter 1: OpenAICompatAdapter

Implements the OpenAI Chat Completions protocol (`POST /v1/chat/completions`).
Covers any provider that speaks this protocol — no code changes needed to add new providers.

```dart
class OpenAICompatAdapter implements ModelRuntime {
  final String baseUrl;      // e.g. "https://api.groq.com/openai/v1"
  final String apiKey;
  final String modelId;      // e.g. "llama-3.3-70b-versatile"
  final String displayName;
}
```

**Pre-configured provider presets** (shown in the provider picker UI):

| UI Label | Base URL | Notes |
|---|---|---|
| Groq | `https://api.groq.com/openai/v1` | Sub-200ms TTFT, free tier: 30 RPM / 14.4K req/day |
| OpenRouter | `https://openrouter.ai/api/v1` | 300+ models, 26+ permanently free at 20 RPM |
| Together AI | `https://api.together.xyz/v1` | Good open-source model selection |
| Fireworks AI | `https://api.fireworks.ai/inference/v1` | Fast serverless inference |
| Home Server (Ollama) | Auto-discovered via LAN scan (port 11434) | Fully local, OpenAI-compat |
| LM Studio | Auto-discovered via LAN scan (port 1234) | Fully local, OpenAI-compat |
| Custom Endpoint | User-provided URL | Any OpenAI-compat server |

**LAN auto-discovery** (for Ollama / LM Studio):
- Scan local subnet for port 11434 (Ollama) and 1234 (LM Studio) on first setup
- Cache discovered addresses; re-scan on demand
- No internet required — pure local network

## Adapter 2: GeminiAdapter

Dedicated adapter for Google Gemini. Uses the Google AI SDK for Android natively where available.

```dart
class GeminiAdapter implements ModelRuntime {
  final String apiKey;
  final GeminiModel model; // flash_lite | flash | pro | nano
}

enum GeminiModel {
  flashLite,   // Gemini 2.0 Flash-Lite — fastest, cheapest
  flash,       // Gemini 2.5 Flash — best balance
  pro,         // Gemini 2.5 Pro — most capable
  nano,        // Gemini Nano — on-device on Pixel 9+, no API key needed
}
```

**Why Gemini gets a dedicated adapter**:
- Proprietary `generateContent` API, not OpenAI-compatible
- Google AI Edge SDK enables on-device Nano inference on Pixel 9+ (no API key, no quota)
- Most generous free tier: 1,000 RPD on Flash-Lite, 50 RPD on Pro
- Native Android SDK with Kotlin/Dart integration path
- Strategic: Google's ecosystem alignment benefits an Android-first app

**On-device Nano path** (Pixel 9+, Galaxy S25+):
- Detect device support via `AIManager.isFeatureSupported()`
- If supported and user enables it: use Nano for fast, free, offline Gemini inference
- Fallback to Flash API if Nano not available

## Adapter 3: AnthropicAdapter

Dedicated adapter for Anthropic Claude via the Messages API.

```dart
class AnthropicAdapter implements ModelRuntime {
  final String apiKey;
  final ClaudeModel model;
}

enum ClaudeModel {
  haiku,    // API model ID: claude-haiku-4-5 — fast, cheapest ($1/$5 per M tokens)
  sonnet,   // API model ID: claude-sonnet-4-6 — best balance
  opus,     // API model ID: claude-opus-4-8 — most capable
}
```

**Why Claude gets a dedicated adapter**:
- Proprietary Messages API (different request/response schema from OpenAI)
- Users request Claude by name more than any other remote model
- Strong tool use capabilities — useful for complex agentic workflows

## Provider UI

The provider picker is a first-class UI screen (not buried in settings).

**Provider buckets shown to the user**:

```
On-device
  ● [Active model name]         currently loaded
  ○ [Other downloaded models]

Home Server
  ○ Ollama (auto-discover)
  ○ LM Studio (auto-discover)

Fast & Free
  ○ Groq — Llama 3.3 70B, DeepSeek R1...
  ○ OpenRouter — 300+ models, 26+ free

Google Gemini
  ○ Gemini Nano (on-device, Pixel only)
  ○ Gemini Flash (free tier: 50 RPD)
  ○ Gemini Pro

Claude (Anthropic)
  ○ Haiku (fastest)
  ○ Sonnet (recommended)
  ○ Opus (most capable)

Custom
  ○ Add custom OpenAI-compatible endpoint...
```

## Credential Storage

All API keys stored in Android Keystore via the `flutter_secure_storage` package.
Keys are never written to disk in plaintext, never included in logs or audit records.

```dart
class CredentialStore {
  Future<void> saveApiKey(String providerId, String apiKey);
  Future<String?> getApiKey(String providerId);
  Future<void> deleteApiKey(String providerId);
}
```

## Unified Provider Config

Stored in SQLite, references credentials by `providerId` (never stores the key inline).

```dart
class ProviderConfig {
  final String id;
  final String providerId;         // "groq", "openrouter", "gemini", "anthropic", "custom"
  final String displayName;
  final String? customBaseUrl;     // only for "custom"
  final String? modelId;           // provider-specific model identifier
  final bool isDefault;
}
```

## Failure Handling

| Scenario | Behavior |
|---|---|
| Network unavailable | Prompt user to switch to on-device model |
| API key invalid (401) | Show credential error, prompt re-entry |
| Rate limit hit (429) | Backoff + retry with user notification |
| Model not available (404) | Fall back to provider's default model, notify user |
| Timeout | Cancel after 30s, offer retry or switch to on-device |
