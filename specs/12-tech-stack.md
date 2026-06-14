# Karmik — Technology Stack

## Overview

Karmik is a Flutter application with a Kotlin native layer for Android system integration.
The inference layer uses llama.cpp compiled for Android ARM64, bridged via Dart FFI.
All persistence is SQLite-based (no external database process required).

## Layer-by-Layer Stack

### UI Layer

| Technology | Version | Purpose |
|---|---|---|
| Flutter | ≥3.22 | Cross-platform UI framework (Android primary) |
| Dart | ≥3.13-dev | Application language (3.13-dev required for dot-shorthand syntax used throughout) |
| Material 3 | — | Design system (dark-first, OLED-optimized) |
| flutter_riverpod | ^2.x | State management |
| go_router | ^14.x | Declarative navigation |
| flutter_animate | ^4.x | Micro-animations |

### Android Native Layer (Kotlin)

Accessed from Dart via Flutter **platform channels**. Each channel is a narrow,
purpose-specific bridge — not a generic JNI dump.

| Channel | Kotlin Class | Purpose |
|---|---|---|
| `karmik/notifications` | `NotificationListenerChannel` | Read, dismiss, reply to notifications |
| `karmik/calendar` | `CalendarChannel` | Read/write Android Calendar Provider |
| `karmik/sms` | `SmsChannel` | Read SMS, send SMS (via SmsManager) |
| `karmik/contacts` | `ContactsChannel` | Read contacts (ContentResolver) |
| `karmik/location` | `LocationChannel` | FusedLocationProviderClient |
| `karmik/launcher` | `LauncherChannel` | Home screen / launcher integration |
| `karmik/service` | `BackgroundServiceChannel` | Control foreground service from Dart |
| `karmik/overlay` | `OverlayChannel` | SYSTEM_ALERT_WINDOW overlay lifecycle |
| `karmik/device` | `DeviceInfoChannel` | RAM, Android API level, NPU check |

### Inference Layer

| Technology | Purpose |
|---|---|
| llama.cpp | Primary local inference (GGUF models, C++, compiled for Android ARM64) |
| Dart FFI | Bridge from Dart to llama.cpp C API |
| MLC-LLM Android AAR | Secondary runtime (better GPU/NPU utilization on Snapdragon) |
| ONNX Runtime Android | Embedding model inference (all-MiniLM-L6-v2 for vector memory) |
| Whisper.cpp | On-device speech-to-text (C++, via Dart FFI, same pattern as llama.cpp) |

**llama.cpp build configuration for Android**:
```cmake
# CMakeLists.txt (android/app/src/main/cpp/)
cmake_minimum_required(VERSION 3.22)
project(karmik_llama)

set(LLAMA_BUILD_TESTS OFF)
set(LLAMA_BUILD_EXAMPLES OFF)
set(GGML_OPENMP OFF)  # Not available on Android
set(GGML_VULKAN ON)   # Enable Vulkan GPU acceleration

add_subdirectory(llama.cpp)
add_library(karmik_llama SHARED karmik_llama_bridge.cpp)
target_link_libraries(karmik_llama llama ggml)
```

Targets: `arm64-v8a` only (primary). `x86_64` for emulator testing.

### Persistence Layer

| Technology | Purpose |
|---|---|
| sqlite_async | Async SQLite access from Dart (non-blocking UI) |
| SQLCipher | AES-256 encryption for all SQLite databases |
| sqlite-vec | Vector similarity search extension (loaded at runtime) |
| flutter_secure_storage | Android Keystore access for encryption keys and API keys |

**Database files** (all in `getApplicationDocumentsDirectory()`):
```
karmik.db          — conversations, agents, tasks, audit log (encrypted)
karmik_memory.db   — long-term memories + vectors (encrypted, sqlite-vec enabled)
karmik_models.db   — model catalog cache, download state (not encrypted — no sensitive data)
```

### Networking

| Technology | Purpose |
|---|---|
| dio | HTTP client (provider adapter requests, plugin HTTP skills, browser.fetch) |
| web_socket_channel | WebSocket support (MCP servers over WebSocket) |

### Background Processing

| Technology | Purpose |
|---|---|
| Android Foreground Service | Always-on orchestration daemon |
| WorkManager | Deferrable, battery-aware scheduled tasks |
| Android AlarmManager | Exact-time schedule triggers (when WorkManager precision is insufficient) |
| NotificationListenerService | Notification access for notification triggers |
| Android Geofencing API | Location-based triggers |

### Plugin Runtime

| Technology | Purpose |
|---|---|
| Dart Isolates | Sandboxed execution of plugin scripts |
| archive (pub.dev) | Unzip plugin packages |

### Build & Tooling

| Technology | Purpose |
|---|---|
| Turbo | Monorepo task runner |
| pnpm | Monorepo package manager (JS tooling) |
| flutter_flavorizr | Build flavors (dev / staging / prod) |
| fastlane | CI/CD automation (build, sign, deploy) |
| GitHub Actions | CI pipeline |

## Monorepo Structure (Stage 1 → Stage 2)

```
karmik/
├── apps/
│   └── mobile/              # Flutter app (the main product)
│       ├── android/
│       │   └── app/src/main/
│       │       ├── cpp/     # llama.cpp + whisper.cpp + bridge code
│       │       └── kotlin/  # Platform channel implementations
│       ├── lib/
│       │   ├── ui/          # Screens, widgets, overlay
│       │   ├── agents/      # Orchestration engine (Stage 1: here)
│       │   ├── runtime/     # Model runtime abstraction (Stage 1: here)
│       │   ├── tools/       # Tool registry (Stage 1: here)
│       │   ├── memory/      # Memory system (Stage 1: here)
│       │   └── plugins/     # Plugin runtime (Stage 1: here)
│       └── pubspec.yaml
├── specs/                   # This folder — design specs
├── package.json             # Turbo config
└── pnpm-lock.yaml
```

**Stage 2 extraction**: `agents/`, `runtime/`, `tools/`, `memory/`, and `plugins/` move into
`packages/` as standalone Dart packages, published to pub.dev as the Karmik SDK.

## Dependency Decisions & Rationale

**Why Flutter over Kotlin-native?**
- Dart FFI enables direct llama.cpp integration without JNI overhead for the inference path
- Faster UI development for the complex overlay + chat + settings surfaces
- Stage 2 SDK can target iOS without rewriting the orchestration layer

**Why llama.cpp over MediaPipe?**
- Broader model support (any GGUF model vs. MediaPipe's curated list)
- Community is larger, updates are faster
- Grammar-constrained decoding (critical for reliable tool call parsing)
- MediaPipe remains available as the `MlcRuntime` fallback for better GPU utilization

**Why SQLite + sqlite-vec over a dedicated vector database?**
- No separate process — everything runs in the same SQLite file
- sqlite-vec is 500KB compiled; dedicated vector DBs (Chroma, Qdrant) require a server process
- Sufficient for the expected scale (thousands of memories, not millions)
- Same encryption story as the rest of the data

**Why Riverpod over Bloc or Provider?**
- Compile-time safety with code generation
- Async-native (fits the inference streaming use case well)
- Testable without BuildContext

## Android Version Support

| Android Version | API Level | Support |
|---|---|---|
| Android 14+ | 34+ | Full (all features) |
| Android 13 | 33 | Full |
| Android 12 | 32 | Full |
| Android 11 | 30 | Full except some foreground service types |
| Android 10 | 29 | Supported (reduced background accuracy) |
| Android 9 and below | <28 | Not supported |

**Minimum SDK**: API 29 (Android 10). Rationale: Vulkan 1.1 (GPU acceleration for llama.cpp)
is available on ~99% of Android 10+ devices. Below API 29, background processing restrictions
make the agent daemon unreliable.

**Target SDK**: API 35 (Android 15). Required for Play Store compliance from August 2025.
