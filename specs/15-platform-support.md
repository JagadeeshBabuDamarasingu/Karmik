# Karmik — Cross-Platform Support

## Overview

Karmik runs on six platforms: Android, iOS, macOS, Windows, Linux, and Web. The
orchestration engine, memory system, plugin runtime, and UI are shared Flutter/Dart code.
Platform-specific capability gaps are handled by feature flags and graceful degradation —
missing features are hidden or replaced with alternatives, never crash.

---

## Feature Matrix

✓ Full support  |  ~ Partial / limited  |  — Not available

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Inference** | | | | | | |
| Local inference (llama.cpp) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| GPU acceleration | Vulkan | Metal | Metal | CUDA/DirectML | CUDA/ROCm | — |
| Remote inference | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Whisper STT (on-device) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| **UI** | | | | | | |
| Floating overlay | ✓ | — | ✓ | ✓ | ~ | — |
| Main app (full UI) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Launcher mode | ✓ | — | — | — | — | — |
| System share target | ✓ | ✓ | ✓ | ✓ | ~ | — |
| **System Integration** | | | | | | |
| Read notifications | ✓ | — | ~ | ~ | ~ | — |
| Dismiss notifications | ✓ | — | — | — | — | — |
| Reply to notifications | ✓ | — | — | — | — | — |
| Calendar read | ✓ | ✓ | ✓ | ~ | ~ | ~ |
| Calendar write | ✓ | ✓ | ✓ | ~ | ~ | — |
| SMS read | ✓ | — | — | — | — | — |
| SMS send | ✓ | — | — | — | — | — |
| Contacts read | ✓ | ✓ | ✓ | ~ | ~ | ~ |
| Location | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Geofence triggers | ✓ | ~ | ~ | ~ | — | — |
| Camera | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Microphone | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Background** | | | | | | |
| Persistent daemon | ✓ | — | ✓ | ✓ | ✓ | — |
| Scheduled triggers | ✓ | ~ | ✓ | ✓ | ✓ | — |
| Notification triggers | ✓ | — | ~ | — | — | — |
| Location triggers | ✓ | ~ | ~ | ~ | — | — |
| On-charging trigger | ✓ | ~ | ✓ | ✓ | ✓ | — |
| **Storage** | | | | | | |
| SQLite + SQLCipher | ✓ | ✓ | ✓ | ✓ | ✓ | ~ |
| sqlite-vec (vectors) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Hardware-backed keys | ✓ | ✓ | ✓ | — | — | — |
| **Privacy** | | | | | | |
| Privacy Mode (OS-level block) | ✓ | ✓ | ✓ | ~ | ~ | — |
| LAN-only mode | ✓ | ✓ | ✓ | ✓ | ✓ | — |

---

## Platform-Specific Notes

### Android (Primary Platform)

Android has the richest system integration of all platforms. All features are supported.

**Unique to Android**:
- `SYSTEM_ALERT_WINDOW` floating bubble that persists across all apps
- `NotificationListenerService` for reading any app's notifications
- SMS read/send
- Launcher mode (replace the home screen)
- Geofencing API with background wake-ups
- WorkManager + ForegroundService for reliable always-on daemon

**Privacy Mode enforcement**: `NetworkSecurityConfig` XML disables cleartext traffic;
a local VPN service intercepts and blocks all outbound connections except an allowlist
of local IP ranges (for LAN-only mode).

---

### iOS

iOS is the most capability-restricted platform due to Apple's sandbox model.

**What's possible**:
- Full local inference via llama.cpp Metal backend (no JIT required — GGUF models work)
- Calendar and contacts via `EventKit` / `CNContactStore`
- Push notifications (deliver results from background tasks)
- Camera and microphone

**What's not possible on iOS**:
- Over-app floating overlay (Apple does not allow `SYSTEM_ALERT_WINDOW`-equivalent)
- Reading other apps' notifications (sandboxing prevents this)
- SMS read/send (no API access)
- Persistent background daemon (iOS kills background processes aggressively)

**iOS alternatives for missing features**:

| Android feature | iOS alternative |
|---|---|
| Floating overlay | **iOS Widget** (WidgetKit, home/lock screen) showing active agent status + quick-reply |
| Floating overlay | **Dynamic Island** integration (iPhone 14 Pro+) showing agent activity |
| Floating overlay | **Action Button** shortcut (iPhone 15 Pro+) to open Karmik instantly |
| Persistent daemon | `BGAppRefreshTask` (30s, OS-scheduled) for lightweight scheduled checks |
| Notification listener | **Notification Center Extension** (reads own notifications only) |
| SMS | **iMessage Business Chat** or user-pasted text (manual) |

**Background constraints**:
- `BGAppRefreshTask`: ~30 seconds, no guarantee of timing, OS schedules when convenient
- `BGProcessingTask`: ~a few minutes, requires device plugged in + WiFi
- Morning Briefing agent fires via `BGProcessingTask` on a "preferred" time hint (OS may
  delay by hours)

**Privacy Mode enforcement**: `NEContentFilter` network extension blocks outbound connections
at the OS level (requires a VPN-style entitlement, approved by Apple).

---

### macOS

macOS offers the most desktop-class system integration of the desktop platforms.

**What's possible**:
- `NSPanel` floating window at `NSWindowLevel.floating` — persists above all app windows
  (equivalent to the Android bubble, but without the circular form factor)
- Calendar and contacts via `EventKit` and `CNContactStore`
- `NSBackgroundActivityScheduler` for periodic background tasks
- `LaunchAgent` (plist in `~/Library/LaunchAgents/`) for persistent daemon that starts on login
- Notification delivery via `UNUserNotificationCenter`
- Notification reading: via macOS Accessibility API (requires "Automation" permission)
  — not as seamless as Android's `NotificationListenerService`
- Metal GPU acceleration for llama.cpp (2× faster than Vulkan on Apple Silicon)

**Distribution**:
- **Mac App Store**: sandboxed build. `NSPanel` overlay works in sandbox. `LaunchAgent`
  daemon does not (sandbox restriction). Calendar and contacts require permission prompt.
- **Direct download (.dmg)**: non-sandboxed. Full daemon support. Recommended for power users.

**macOS overlay (NSPanel) UX**:
```
macOS floating panel (top-right corner by default, user-draggable):
┌─────────────────────────────────────────┐
│ ◆ Karmik   Aria ▾   [🎤]  [⚙]  [−]   │
│─────────────────────────────────────────│
│                                         │
│  Type or speak...                       │
│                                         │
│─────────────────────────────────────────│
│  [Streaming response appears here]      │
└─────────────────────────────────────────┘
```

Keyboard shortcut to show/hide: user-configurable (default: `⌥Space`).

---

### Windows

**What's possible**:
- `WS_EX_LAYERED | WS_EX_TOPMOST` window for the floating overlay
- Notification delivery via `Windows.UI.Notifications` (WinRT toast notifications)
- Calendar: CalDAV client (user provides a CalDAV URL, e.g., Google Calendar, Outlook)
  — no native Win32 calendar API; UWP Calendar API is available but complex
- Windows Service for persistent daemon (starts on boot, runs as local user)
- Task Scheduler for scheduled triggers
- Location via `Windows.Devices.Geolocation`
- Contacts: limited (Windows People app via `Windows.ApplicationModel.Contacts`)
- CUDA acceleration for NVIDIA GPUs; DirectML fallback for all DirectX 12 GPUs

**What's not available**:
- SMS (Windows does not expose SMS from paired Android phones to third-party apps in API)
- Notification reading (no equivalent of `NotificationListenerService`)
- Launcher mode (Windows doesn't support replacing the shell in the same way)

**Windows overlay UX**: similar to macOS panel. Default position: bottom-right corner.
Triggered via system tray icon double-click or configurable hotkey (default: `Ctrl+Shift+K`).

**Privacy Mode enforcement**: `WinSockProvider` layered service provider or Windows
Filtering Platform (WFP) callout. Simpler fallback: replace `HttpClient` factory with a
blocking stub at the Dart layer.

---

### Linux

**What's possible**:
- Floating overlay via `wlr-layer-shell` (Wayland compositors: Sway, Hyprland, Wayfire) or
  `_NET_WM_WINDOW_TYPE_DOCK` (X11, all desktop environments)
- `systemd` user service for persistent daemon (`~/.config/systemd/user/karmik.service`)
- `libnotify` for notification delivery (D-Bus `org.freedesktop.Notifications`)
- Calendar and contacts via Evolution Data Server (if installed) or CalDAV
- Location via `geoclue2` (D-Bus)
- CUDA acceleration (NVIDIA), ROCm (AMD), or AVX2 CPU fallback
- `libsecret` for credential storage (integrates with GNOME Keyring, KWallet)

**Wayland overlay note**: `wlr-layer-shell` is a wlroots extension protocol; it works on
Sway, Hyprland, and Wayfire. On GNOME Wayland (Mutter), the protocol is not available —
fallback is a regular `WL_SHELL` popup window that loses focus when the user clicks elsewhere.
Recommend: show a non-floating compact panel mode as fallback on GNOME Wayland.

**Distribution**:
- `.deb` (Debian/Ubuntu)
- `.rpm` (Fedora/RHEL)
- Flatpak (sandboxed — `libsecret`, systemd service, and overlay may require portal permissions)
- AppImage (portable, no install)

**Privacy Mode enforcement**: `iptables`/`nftables` OUTPUT rule scoped to Karmik's UID,
added when Privacy Mode is enabled. Requires `CAP_NET_ADMIN` or `sudo` on first setup —
show an OS permission dialog explaining the requirement.

---

### Web

Web is a remote-inference-only, limited-feature client. Its primary purpose is:
1. Access Karmik from any device without installing an app
2. Manage agents, view history, and configure settings remotely
3. Bridge use cases (e.g., share a webpage to Karmik from a desktop browser)

**What's possible on Web**:
- Full agent chat UI (via remote inference)
- Memory browser and search
- Agent configuration
- Audit log viewer
- Notification delivery via Web Notifications API
- Location via `navigator.geolocation`
- Camera + microphone (via WebRTC `getUserMedia`)

**What's not available on Web**:
- Local inference (no llama.cpp WASM in production; too slow, too large)
- Floating overlay (browser tab only)
- System notifications reading (browsers cannot read other apps' notifications)
- SMS, calendar write, contacts write
- Background daemon (Service Workers can do limited sync but not long inference)
- sqlite-vec (WASM SQLite extension loading is not supported cross-browser)

**Web storage**: IndexedDB via `sqflite_web`. Conversations and memories are stored
per-origin. Encryption uses AES-GCM with a key derived from a user-provided passphrase
(entered on login). Without a passphrase, data is stored unencrypted — a warning is shown.

**Web-only features**:
- **Shared link to conversation**: generate a shareable URL for a conversation (exported
  as a static HTML page, no server required — data embedded in the URL hash)
- **Browser extension (Stage 2)**: Chrome/Firefox extension to capture web content directly
  to Karmik without navigating to the web app

---

## Platform-Adaptive UI

The Flutter UI adapts its layout and interaction patterns per platform class:

### Mobile (Android, iOS)
- Bottom navigation, large touch targets (48dp+)
- Floating action button for new agent/message
- Swipe gestures for navigation and overlay control

### Desktop (macOS, Windows, Linux)
- Sidebar navigation (persistent on wide screens)
- Keyboard shortcuts throughout
- Right-click context menus
- Resizable panels
- Drag-and-drop files into the chat input
- Dense information density (smaller font size, tighter spacing)

### Web
- Responsive: mobile layout on narrow viewports, desktop layout on wide
- No keyboard shortcuts that conflict with browser defaults
- Tab key navigable

### Overlay (Android, macOS, Windows, Linux-Wayland)

All overlay implementations share the same UX states (Idle, Peek, Expanded) from
`specs/10-ui.md`. The visual form factor differs:

| Platform | Idle state | Expanded state |
|---|---|---|
| Android | 56dp circular bubble at screen edge | Bottom-anchored chat panel (60% screen height) |
| macOS | Menu bar icon (◆) | Floating `NSPanel` (top-right corner) |
| Windows | System tray icon | Floating Win32 panel (bottom-right corner) |
| Linux (Wayland) | Dock/layer-shell icon | Layer-shell panel at screen edge |
| Linux (X11) | System tray icon (`_NET_WM_WINDOW_TYPE_DOCK`) | Floating panel |

On platforms where an overlay isn't possible (iOS, Web), the equivalent entry point is:
- **iOS**: home screen widget / Dynamic Island / Action Button → opens main app
- **Web**: pinned browser tab, browser notifications

---

## Platform-Specific Permissions Summary

### Android
`SYSTEM_ALERT_WINDOW`, `BIND_NOTIFICATION_LISTENER_SERVICE`, `READ_CALENDAR`,
`WRITE_CALENDAR`, `READ_CONTACTS`, `READ_SMS`, `SEND_SMS`, `ACCESS_FINE_LOCATION`,
`CAMERA`, `RECORD_AUDIO`, `FOREGROUND_SERVICE`, `POST_NOTIFICATIONS`,
`RECEIVE_BOOT_COMPLETED`, `SCHEDULE_EXACT_ALARM`

### iOS
`NSCalendarsUsageDescription` (EventKit), `NSContactsUsageDescription`, `NSLocationWhenInUseUsageDescription`,
`NSCameraUsageDescription`, `NSMicrophoneUsageDescription`, `BGTaskScheduler` entitlement,
`com.apple.developer.usernotifications.time-sensitive` (for time-sensitive notifications)

### macOS
`com.apple.security.network.client` (outbound network), `com.apple.security.personal-information.calendars`,
`com.apple.security.personal-information.addressbook`, `NSLocationUsageDescription`,
`NSCameraUsageDescription`, `NSMicrophoneUsageDescription`,
Accessibility API access (for notification reading — user-granted in System Settings)

### Windows
No formal permission manifest; OS prompts at first use for: microphone, camera, location.
Windows Service registration requires elevation on first setup.

### Linux
`CAP_NET_ADMIN` for Privacy Mode iptables rules (prompted once).
`geoclue2` access for location (D-Bus policy).
All other capabilities are user-space, no special permissions required.

### Web
Web Notifications API permission, `getUserMedia` permission (camera/microphone),
`navigator.geolocation` permission — all prompted by the browser at first use.
