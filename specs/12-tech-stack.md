# Karmik — Technology Stack

## Overview

Karmik is a Flutter application that runs on Android, iOS, macOS, Windows, Linux, and Web.
The orchestration engine, memory system, and plugin runtime are shared Dart code across all
platforms. Platform-specific native layers (Kotlin, Swift, Win32, GTK, JS) handle system
integration features that vary by OS.

The inference layer uses llama.cpp compiled per target architecture, bridged via Dart FFI.
On Web, local inference is not available; remote providers are the only inference option.
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

### Platform Native Layers

Each platform implements the same set of **platform channels** in the OS-native language.
The Dart side calls a unified abstract interface; the native side provides the platform-specific
implementation. Channels that have no equivalent on a platform return `unsupported` gracefully.

#### Android (Kotlin)

| Channel | Implementation | Purpose |
|---|---|---|
| `karmik/notifications` | `NotificationListenerService` | Read, dismiss, reply to notifications |
| `karmik/calendar` | `CalendarProvider` (ContentResolver) | Read/write Android Calendar |
| `karmik/sms` | `SmsManager` / `ContentResolver` | Read SMS, send SMS |
| `karmik/contacts` | `ContactsContract` (ContentResolver) | Read contacts |
| `karmik/location` | `FusedLocationProviderClient` | Location + geofence |
| `karmik/launcher` | `LauncherApps` + home screen intent | Launcher mode |
| `karmik/service` | `ForegroundService` + `WorkManager` | Background daemon control |
| `karmik/overlay` | `SYSTEM_ALERT_WINDOW` | Floating bubble |
| `karmik/device` | `ActivityManager`, `Build` | RAM, API level, NPU check |

#### iOS / macOS (Swift)

| Channel | Implementation | Purpose |
|---|---|---|
| `karmik/notifications` | `UNUserNotificationCenter` (iOS: read own; macOS: broader via accessibility) | Notification delivery + limited read |
| `karmik/calendar` | `EventKit` (EKEventStore) | Read/write calendars |
| `karmik/sms` | Not available (iOS/macOS sandboxing) | Unsupported |
| `karmik/contacts` | `CNContactStore` | Read contacts |
| `karmik/location` | `CoreLocation` | Location; geofence via `CLCircularRegion` |
| `karmik/launcher` | Not applicable | Unsupported |
| `karmik/service` | iOS: `BGTaskScheduler`; macOS: `NSBackgroundActivityScheduler` + LaunchAgent | Background tasks |
| `karmik/overlay` | iOS: not available (no over-app overlay); macOS: `NSPanel` (floating, `NSWindowLevel.floating`) | macOS: floating panel; iOS: lock screen widget / Shortcuts action |
| `karmik/device` | `ProcessInfo`, `os_proc_available_memory()` | RAM, OS version |

#### Windows (C++ / Win32)

| Channel | Implementation | Purpose |
|---|---|---|
| `karmik/notifications` | `Windows.UI.Notifications` (WinRT) | Deliver + limited read via Phone Link bridge |
| `karmik/calendar` | CalDAV client (no native Windows calendar API without UWP) | Calendar via user-provided CalDAV URL |
| `karmik/sms` | Not available | Unsupported |
| `karmik/contacts` | `Windows.ApplicationModel.Contacts` (UWP) or local file | Read contacts |
| `karmik/location` | `Windows.Devices.Geolocation` (WinRT) | Location; geofence via `Geovisit` |
| `karmik/launcher` | Not applicable | Unsupported |
| `karmik/service` | Windows Service + Task Scheduler (`schtasks`) | Background daemon |
| `karmik/overlay` | `WS_EX_LAYERED \| WS_EX_TOPMOST` Win32 window | Floating overlay |
| `karmik/device` | `GlobalMemoryStatusEx`, `GetSystemInfo` | RAM, architecture |

#### Linux (C / GTK)

| Channel | Implementation | Purpose |
|---|---|---|
| `karmik/notifications` | `libnotify` (D-Bus `org.freedesktop.Notifications`) | Deliver notifications |
| `karmik/calendar` | Evolution Data Server (`libecal`) or CalDAV | Calendar |
| `karmik/sms` | Not available | Unsupported |
| `karmik/contacts` | Evolution Data Server (`libebook`) | Contacts |
| `karmik/location` | `geoclue2` (D-Bus) | Location |
| `karmik/launcher` | Not applicable | Unsupported |
| `karmik/service` | `systemd` user service (`libsystemd` D-Bus activation) | Background daemon |
| `karmik/overlay` | X11: `_NET_WM_WINDOW_TYPE_DOCK`; Wayland: `wlr-layer-shell` protocol | Floating overlay (compositor-dependent) |
| `karmik/device` | `/proc/meminfo`, `uname` | RAM, architecture |

#### Web (JavaScript / WASM)

| Channel | Implementation | Purpose |
|---|---|---|
| `karmik/notifications` | Web Notifications API | Deliver notifications (requires permission) |
| `karmik/calendar` | CalDAV client (user provides URL) | Calendar |
| `karmik/sms` | Not available | Unsupported |
| `karmik/contacts` | Contact Picker API (Chrome only) | Limited read |
| `karmik/location` | `navigator.geolocation` | Location |
| `karmik/launcher` | Not applicable | Unsupported |
| `karmik/service` | Service Worker + Background Sync API | Limited background (no long inference) |
| `karmik/overlay` | Not available (browser tab only) | Unsupported |
| `karmik/device` | `navigator.deviceMemory`, `navigator.hardwareConcurrency` | Approximate RAM |

**Web inference**: llama.cpp via WebAssembly is too slow for production use. On Web, only
remote providers are available for inference. The model manager shows an info message:
"Local model inference is not available in the browser. Connect a remote provider to continue."

### Inference Layer

| Technology | Purpose |
|---|---|
| llama.cpp | Primary local inference (GGUF models, C++, compiled per platform) |
| Dart FFI | Bridge from Dart to llama.cpp C API (all platforms except Web) |
| MLC-LLM | Secondary runtime — better GPU utilization on supported hardware |
| ONNX Runtime | Embedding model inference (all-MiniLM-L6-v2); available all platforms except Web |
| Whisper.cpp | On-device speech-to-text (C++, via Dart FFI) |

**llama.cpp compile targets and GPU backends**:

| Platform | Architecture | GPU backend | Notes |
|---|---|---|---|
| Android | `arm64-v8a` | Vulkan | Primary target |
| Android (emulator) | `x86_64` | None (CPU only) | Testing only |
| iOS | `arm64` | Metal (via llama.cpp Metal backend) | App Store: no JIT, but llama.cpp uses AOT |
| macOS (Apple Silicon) | `arm64` | Metal | Best performance; ~2× faster than Vulkan |
| macOS (Intel) | `x86_64` | None (CPU, AVX2) | Slower; Intel Macs are secondary |
| Windows | `x86_64` | CUDA (optional), DirectML, AVX2 CPU | CUDA if NVIDIA GPU present |
| Linux | `x86_64` | CUDA (optional), ROCm (AMD), AVX2 CPU | Detect at runtime |
| Web | — | — | No local inference; remote only |

**CMake build flags per platform**:
```cmake
# Android
set(GGML_VULKAN ON)
set(GGML_OPENMP OFF)

# iOS / macOS
set(GGML_METAL ON)
set(GGML_OPENMP OFF)  # Not reliable on Apple platforms

# Windows (CUDA path, optional)
set(GGML_CUDA ON)  # if CUDA toolkit detected
set(GGML_AVX2 ON)

# Linux
set(GGML_CUDA ON)   # if libcuda.so present
set(GGML_ROCM ON)   # if ROCm present (mutually exclusive with CUDA)
set(GGML_AVX2 ON)
```

### Persistence Layer

| Technology | Purpose |
|---|---|
| sqlite_async | Async SQLite access from Dart (non-blocking UI; all platforms) |
| SQLCipher | AES-256 encryption for all SQLite databases (all platforms) |
| sqlite-vec | Vector similarity search extension (all platforms except Web) |
| flutter_secure_storage | Platform-native credential storage (see table below) |

**Platform credential storage** (`flutter_secure_storage` backend):

| Platform | Backend | Hardware-backed |
|---|---|---|
| Android | Android Keystore | Yes (StrongBox on Pixel/flagship) |
| iOS | iOS Keychain | Yes (Secure Enclave) |
| macOS | macOS Keychain | Yes (Secure Enclave on Apple Silicon) |
| Windows | Windows Credential Manager (DPAPI) | No (software encryption) |
| Linux | `libsecret` / Secret Service (D-Bus) | Depends on distro (KeePass, GNOME Keyring) |
| Web | `window.sessionStorage` (in-memory, cleared on tab close) | No — API keys not persisted on Web |

**Web persistence**: `sqflite_web` (IndexedDB backend). SQLCipher is not available in the
browser; the Web build uses AES-GCM encryption in WASM with a key derived from a session
passphrase the user enters on each visit. Data is stored per-origin in IndexedDB.

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

| Technology | Platform | Purpose |
|---|---|---|
| Android ForegroundService | Android | Always-on orchestration daemon |
| Android WorkManager | Android | Deferrable, battery-aware scheduled tasks |
| Android AlarmManager | Android | Exact-time schedule triggers |
| NotificationListenerService | Android | Notification access for notification triggers |
| Android Geofencing API | Android | Location-based triggers |
| `BGTaskScheduler` (BGAppRefreshTask) | iOS | Short background refresh (≤30s, OS-scheduled) |
| `BGProcessingTask` | iOS | Longer background tasks (requires charging, WiFi) |
| `NSBackgroundActivityScheduler` | macOS | Periodic background tasks |
| LaunchAgent (`launchd`) | macOS | Persistent daemon for always-on orchestration |
| Windows Service | Windows | Persistent background daemon |
| Task Scheduler (`schtasks`) | Windows | Scheduled triggers |
| `systemd` user service | Linux | Persistent daemon + scheduled triggers |

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
│   └── karmik/              # Single Flutter app (all platforms share this codebase)
│       ├── android/
│       │   └── app/src/main/
│       │       ├── cpp/        # llama.cpp + whisper.cpp + bridge code (shared C++)
│       │       └── kotlin/     # Android platform channel implementations
│       ├── ios/
│       │   └── Runner/
│       │       └── swift/      # iOS platform channel implementations
│       ├── macos/
│       │   └── Runner/
│       │       └── swift/      # macOS platform channel implementations
│       ├── windows/
│       │   └── runner/
│       │       └── cpp/        # Windows platform channel implementations (Win32)
│       ├── linux/
│       │   └── runner/
│       │       └── cpp/        # Linux platform channel implementations (GTK + D-Bus)
│       ├── web/
│       │   └── index.html      # Web entry point (no native channels)
│       ├── lib/
│       │   ├── ui/             # Screens, widgets, overlay (adaptive per platform)
│       │   ├── agents/         # Orchestration engine
│       │   ├── runtime/        # Model runtime abstraction
│       │   ├── tools/          # Tool registry
│       │   ├── memory/         # Memory system
│       │   └── plugins/        # Plugin runtime
│       └── pubspec.yaml
├── specs/                   # Design specs
├── package.json             # Turbo config
└── pnpm-lock.yaml
```

**Shared C++ layer**: `android/app/src/main/cpp/` contains the llama.cpp and whisper.cpp
sources. iOS, macOS, Windows, and Linux builds reference the same source tree via CMake
(each platform invokes cmake with its own toolchain file). The C++ bridge API is identical;
only the CMake flags and GPU backend differ.

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

## Platform Version Support

### Android

| Version | API Level | Support |
|---|---|---|
| Android 14+ | 34+ | Full |
| Android 13 | 33 | Full |
| Android 12 | 32 | Full |
| Android 11 | 30 | Full except some foreground service types |
| Android 10 | 29 | Supported (reduced background accuracy) |
| < Android 10 | <29 | Not supported |

Minimum: API 29. Target: API 35.

### iOS

| Version | Support |
|---|---|
| iOS 17+ | Full |
| iOS 16 | Full |
| iOS 15 | Supported (BGTaskScheduler available since iOS 13) |
| < iOS 15 | Not supported |

Minimum: iOS 15. Overlay is not available on iOS (OS restriction). Background tasks are
limited to ~30s (BGAppRefreshTask) or ~a few minutes when plugged in (BGProcessingTask).
Local inference supported via llama.cpp Metal backend.

### macOS

| Version | Support |
|---|---|
| macOS 14 (Sonoma)+ | Full (floating NSPanel overlay) |
| macOS 13 (Ventura) | Full |
| macOS 12 (Monterey) | Supported (no Metal 3, slightly slower inference) |
| < macOS 12 | Not supported |

Minimum: macOS 12. Distributed via Mac App Store and direct `.dmg` download.
The Mac App Store build runs in a sandbox — the `NSPanel` overlay and `LaunchAgent` daemon
require the direct download (non-sandboxed) distribution.

### Windows

| Version | Support |
|---|---|
| Windows 11 | Full |
| Windows 10 (21H2+) | Full |
| Windows 10 (< 21H2) | Supported (WinRT APIs have reduced feature set) |
| Windows 8/7 | Not supported |

Minimum: Windows 10. Distributed as an MSIX package (Microsoft Store) and standalone installer.

### Linux

| Distribution | Support |
|---|---|
| Ubuntu 22.04+ | Full (GTK 3, Wayland + X11) |
| Debian 12+ | Full |
| Fedora 39+ | Full |
| Arch Linux | Supported (rolling release) |
| Other distros | Best-effort (requires GTK 3.24+, `libsecret`, `geoclue2`) |

Distributed as a `.deb`, `.rpm`, Flatpak, and AppImage. Flatpak is the recommended
distribution method for best sandbox compatibility.

### Web

| Browser | Support |
|---|---|
| Chrome / Chromium 120+ | Full |
| Firefox 120+ | Full (Contact Picker API not available — contacts disabled) |
| Safari 17+ | Full |
| Edge 120+ | Full |

Web is remote-inference-only. No local model execution. Storage is IndexedDB
(encrypted with session passphrase). Service Worker provides limited offline caching of the
app shell, but agent sessions require a network connection to the remote provider.
