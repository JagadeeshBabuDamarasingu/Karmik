# Karmik — Productivity Tools

## Overview

This spec covers the productivity tool categories that extend Karmik beyond the core
agent/calendar/task/notification capabilities. These tools make Karmik a genuine daily
driver by covering email, the clipboard, screen capture, focus sessions, weather, health,
reading lists, notes, and translation.

All tools follow the same `Tool` interface from `specs/07-tool-registry.md`. Platform
availability and Privacy Mode behavior are noted per category.

---

## Email Tools

### Platform Integration

| Platform | Backend |
|---|---|
| Android | Android `ContentResolver` (Gmail/other email apps) + Gmail REST API (OAuth2) |
| iOS | `MessageUI` framework + IMAP fallback |
| macOS | `MailKit` framework + IMAP/SMTP |
| Windows | IMAP/SMTP (no native Win32 email API without UWP) |
| Linux | IMAP/SMTP |
| Web | IMAP/SMTP via server-side proxy (not direct browser access) |

**Universal fallback**: generic IMAP4 (RFC 3501) for read + SMTP (RFC 5321) for send.
Credentials (server, port, username, password/app-password) stored in platform keystore.

**OAuth2 providers** (preferred when available; no password stored):
- Gmail: `com.google.android.gms` OAuth2 on Android; Google OAuth2 web flow on other platforms
- Microsoft 365 / Outlook: MSAL OAuth2
- iCloud Mail: app-specific password (Apple does not expose OAuth2 for IMAP)

### Tool Definitions

```
email.list
  → List emails in a folder, most recent first
  Input: { "folder"?: "inbox|sent|drafts|all", "from"?: string, "since"?: ISO8601,
           "unreadOnly"?: bool, "limit"?: int }
  Output: [{ "id", "subject", "from", "to": [], "date": ISO8601, "snippet", "read": bool,
             "hasAttachments": bool }]

email.read
  → Read the full content of an email
  Input: { "id": string }
  Output: { "id", "subject", "from", "to": [], "cc": [], "date": ISO8601,
            "body": string, "bodyHtml"?: string, "attachments": [{ "name", "size", "mimeType" }] }

email.search
  → Search emails by query string
  Input: { "query": string, "folder"?: string, "limit"?: int }
  Output: same as email.list

email.compose
  → Compose and save a draft (does NOT send)
  Input: { "to": string[], "subject": string, "body": string, "cc"?: string[], "bcc"?: string[] }
  Output: { "draftId": string }

email.send
  → Send a composed email (requires Co-pilot confirmation)
  Input: { "to": string[], "subject": string, "body": string, "cc"?: string[], "bcc"?: string[] }
  Output: { "sent": bool, "messageId": string }
  Co-pilot: always — shows full email preview before sending

email.reply
  → Reply to an existing email (requires Co-pilot confirmation)
  Input: { "id": string, "body": string, "replyAll"?: bool }
  Output: { "sent": bool }
  Co-pilot: always

email.archive
  → Archive an email (moves out of inbox)
  Input: { "id": string }
  Output: { "archived": bool }

email.label
  → Add or remove a label/folder tag
  Input: { "id": string, "label": string, "remove"?: bool }
  Output: { "updated": bool }
```

**Privacy Mode**: IMAP/SMTP to a self-hosted server (LAN IP) — allowed in Privacy Mode.
Gmail API, Microsoft Graph, cloud IMAP — blocked in Privacy Mode.

**Permission**: `tools.email.*` off by default. `email.send` and `email.reply` additionally
require `requiresCopilot: true` enforced in `PermissionChecker`.

**Account configuration**: Settings → Email → Add account (type: Gmail / Outlook / iCloud /
IMAP+SMTP). Multiple accounts supported; each `email.*` tool call can specify
`accountId?: string` to target a specific account.

---

## Clipboard Tools

```
clipboard.read
  → Read the current clipboard content
  Input: {}
  Output: { "text"?: string, "imagePath"?: string, "mimeType": string }
  Note: returns text OR image path (image saved to temp file), not both simultaneously

clipboard.write
  → Write text to the clipboard
  Input: { "text": string }
  Output: { "written": bool }
```

**Platform**: Android `ClipboardManager`; iOS `UIPasteboard`; macOS `NSPasteboard`;
Windows `OpenClipboard`/`SetClipboardData`; Linux `xclip` (X11) / `wl-clipboard` (Wayland).

**Privacy**: entirely local — no data leaves the device. Works in Privacy Mode.

**Permission**: `tools.clipboard.*` off by default. Users are warned that clipboard may
contain sensitive data (passwords, credit cards) when granting this permission.

**Audit log**: `clipboard.read` logs "Read clipboard content (N chars)" — never the
actual clipboard text (following the same sanitization principle as notifications).

**Natural use cases**:
- "Summarize what I just copied"
- "Fix the bug in the code on my clipboard"
- "Translate the text I copied"
- "Write this email draft to my clipboard so I can paste it"

---

## Screen Capture + OCR Tools

```
screen.capture
  → Take a screenshot of the current screen or a specified region
  Input: { "region"?: { "x": int, "y": int, "width": int, "height": int } }
  Output: { "imagePath": string, "width": int, "height": int }
  Platform: Android (MediaProjection), macOS (CGWindowListCreateImage), Windows (PrintWindow / GDI+),
            Linux (scrot / grim). NOT available on iOS (sandboxing prevents cross-app capture) or Web.

screen.ocr
  → Extract text from an image file or the live screen
  Input: { "imagePath"?: string }  // null = capture screen first, then OCR
  Output: { "text": string, "confidence": float, "blocks": [{ "text", "bounds": {} }] }
```

**OCR backends** (in priority order):
1. **Platform native** (preferred, zero download):
   - Android: ML Kit Text Recognition (bundled with Google Play Services)
   - iOS/macOS: Vision framework (`VNRecognizeTextRequest`)
   - Windows: `Windows.Media.Ocr` (WinRT)
   - Linux: Tesseract.cpp (bundled, ~30MB, Dart FFI)
2. **Tesseract.cpp** (fallback on non-platform paths, and primary on Linux)

**Privacy**: entirely on-device. Works in Privacy Mode.

**Android permission**: `MediaProjection` API requires the user to grant "screen recording"
permission each session (Android security requirement — cannot be pre-granted in manifest).
A clear consent dialog is shown explaining why the agent needs screen access.

**Co-pilot note**: `screen.capture` is not a Co-pilot tool by default, but agents should
prefer to ask the user before capturing (the overlay is visible so the user can see
capture is happening). Background agents cannot call `screen.capture` (Autopilot mode
restriction — capturing screen without user awareness is a privacy violation).

---

## Focus Mode Tools

Manages timed focus sessions with platform-level distraction blocking.

```
focus.start
  → Start a focus session
  Input: { "durationMinutes": int, "label"?: string, "blockNotifications"?: bool }
  Output: { "sessionId": string, "endsAt": ISO8601 }

focus.status
  → Get the current focus session state
  Input: {}
  Output: { "active": bool, "sessionId"?: string, "label"?: string,
            "minutesRemaining"?: int, "completedToday": int }

focus.end
  → End the current session early
  Input: {}
  Output: { "ended": bool, "minutesCompleted": int }
```

**DND / notification blocking per platform**:
- Android: `NotificationManager.setInterruptionFilter(INTERRUPTION_FILTER_NONE)` or
  `INTERRUPTION_FILTER_PRIORITY` (calls + priority contacts only). Requires
  `ACCESS_NOTIFICATION_POLICY` permission.
- iOS: `Focus` framework (iOS 15+) — activates a custom "Karmik Focus" mode via
  `INFocusStatusCenter`. Requires user to allow Karmik to control Focus in iOS Settings.
- macOS: Sets "Do Not Disturb" via `NSUserDefaults` shared suite (requires Accessibility
  permission on macOS 12+; workaround via Shortcuts on macOS 13+).
- Windows: Focus Assist via `Windows.UI.Shell.FocusSessionManager` (WinRT).
- Linux: GNOME: `gsettings set org.gnome.desktop.notifications show-banners false`;
  KDE: `dbus-send` to `org.kde.plasmashell`; PipeWire/PulseAudio for audio muting.

**Focus session storage**: sessions logged in `karmik.db` with start time, end time,
planned duration, and label. Used by the Focus Coach agent for streaks and stats.

**Permission**: `tools.focus.*` off by default. Requires notification policy permission.

---

## Weather Tools

```
weather.current
  → Current weather conditions at the device's location (or a specified location)
  Input: { "location"?: string, "units"?: "metric|imperial" }
  Output: { "temperature": float, "feelsLike": float, "humidity": int, "windSpeed": float,
            "windDirection": string, "conditions": string, "uv": float, "visibility": float,
            "location": { "city", "country" } }

weather.forecast
  → Weather forecast
  Input: { "days": int, "location"?: string, "units"?: "metric|imperial" }
  Output: [{ "date": ISO8601, "high": float, "low": float, "conditions": string,
             "precipProbability": float, "hourly": [{ "time", "temp", "conditions" }] }]
```

**Implementation**: [Open-Meteo](https://open-meteo.com/) API.
- Free, no API key required
- Privacy-safe: only GPS coordinates are sent (no user account, no tracking)
- Sends: latitude, longitude, unit preference
- Receives: weather data in JSON
- Local cache: 1-hour TTL per location in `karmik.db`

**Privacy Mode**: Open-Meteo is a public internet API — blocked in Privacy Mode.
In Privacy Mode, `weather.*` returns `success: false` with message: "Weather data requires
internet access. Disable Privacy Mode or use LAN-only mode to access weather."

**Platform**: works on all platforms that have location access or accept a city name string.

---

## Health & Fitness Tools

```
health.steps
  → Step count for a date range
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: { "totalSteps": int, "daily": [{ "date": ISO8601, "steps": int }] }

health.sleep
  → Sleep data for recent nights
  Input: { "nights": int }
  Output: [{ "date": ISO8601, "bedtime": ISO8601, "wakeTime": ISO8601,
             "durationHours": float, "deepSleepHours"?: float, "quality"?: string }]

health.activity
  → Workouts and active minutes
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: [{ "date": ISO8601, "type": string, "durationMinutes": int,
             "calories"?: int, "distance"?: float }]

health.heart_rate
  → Heart rate data
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: { "restingBpm"?: int, "samples": [{ "time": ISO8601, "bpm": int }] }
```

**Platform integration**:
- **Android**: Android Health Connect (`androidx.health.connect.client`). Requires
  `android.permission.health.READ_STEPS`, `.READ_SLEEP`, etc. per data type.
  Health Connect is available on Android 9+ (API 28+) via Health Connect APK or
  Android 14+ (built-in).
- **iOS**: Apple HealthKit (`HKHealthStore`). Permission requested per data type via
  `requestAuthorization`. Privacy-sensitive — user sees exactly which types Karmik reads.
- **Desktop / Web**: not available. Returns `{ success: false, error: "Health data not available on this platform." }`

**Privacy**: health data is strictly on-device. Never sent anywhere. Works in Privacy Mode.
Health data is **not** stored in Karmik's own databases — it's read live from the platform
health store each time a tool is called. Karmik does not cache health metrics.

**Audit log**: health tool calls are logged with the date range queried, never the values
themselves. Example: "Read step data for 2026-06-01 through 2026-06-14."

---

## Reading List + Web Clipper

Karmik's built-in reading list saves articles with AI summaries and enables agents to
surface and digest content proactively.

```
readinglist.add
  → Save a URL to the reading list with an AI-generated summary
  Input: { "url": string, "tags"?: string[], "note"?: string }
  Output: { "id": string, "title": string, "summary": string }
  Note: fetches URL content via browser.fetch, generates summary via inference,
        embeds summary for semantic search

readinglist.list
  → List saved items
  Input: { "tag"?: string, "unreadOnly"?: bool, "limit"?: int, "query"?: string }
  Output: [{ "id", "url", "title", "summary", "tags": [], "savedAt": ISO8601, "read": bool }]

readinglist.read
  → Fetch and return the full article text of a saved item
  Input: { "id": string }
  Output: { "id", "url", "title", "content": string, "wordCount": int }

readinglist.mark_read
  → Mark an item as read
  Input: { "id": string }
  Output: { "updated": bool }

readinglist.delete
  → Remove an item
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: new `reading_list` table in `karmik.db`. Summary and embedding stored at
add time. Full article text stored if under 100KB; otherwise, only the summary is kept.

**Semantic search**: `readinglist.list { "query": "..." }` runs a vector similarity search
over the embeddings (same pipeline as `memory.search`).

**Privacy**: article URLs and content stay on-device. `readinglist.add` calls `browser.fetch`
which is a network request — blocked in Privacy Mode.

**Share sheet integration**: when another app shares a URL to Karmik (see `specs/10-ui.md`
Surface 4), one of the quick action options is "Save to reading list" — triggers
`readinglist.add` without opening an agent chat.

---

## Note-Taking Tools

Karmik can read from and write to the user's existing note-taking tool, treating it as
structured persistent storage distinct from Karmik's own memory system.

```
notes.create
  → Create a new note
  Input: { "title": string, "content": string, "folder"?: string, "tags"?: string[] }
  Output: { "id": string, "url"?: string }

notes.read
  → Read a note by ID or title
  Input: { "id"?: string, "title"?: string }
  Output: { "id", "title", "content": string, "tags": [], "modifiedAt": ISO8601 }

notes.list
  → List recent notes
  Input: { "folder"?: string, "tag"?: string, "limit"?: int }
  Output: [{ "id", "title", "snippet", "tags": [], "modifiedAt": ISO8601 }]

notes.search
  → Search notes by full-text query
  Input: { "query": string, "limit"?: int }
  Output: same as notes.list

notes.append
  → Append text to an existing note
  Input: { "id": string, "content": string }
  Output: { "updated": bool }

notes.update
  → Replace the full content of a note
  Input: { "id": string, "title"?: string, "content"?: string, "tags"?: string[] }
  Output: { "updated": bool }
```

**Supported backends** (user picks in Settings → Notes → Note provider):

| Backend | Platform | Local/Cloud | Privacy Mode |
|---|---|---|---|
| **Markdown files** (default) | All | Local (`~/Documents/karmik-notes/`) | ✓ |
| **Obsidian vault** | All | Local (user specifies vault path) | ✓ |
| **Apple Notes** | iOS / macOS | Local + iCloud sync | ✓ (local read/write) |
| **Notion** | All | Cloud (Notion API) | ✗ |
| **Joplin** (local) | macOS / Linux / Windows | Local (Joplin data dir) | ✓ |
| **Joplin** (sync) | All | Cloud | ✗ |

The **Markdown files** backend uses `files.*` tools under the hood — each note is a `.md`
file. `notes.search` uses SQLite FTS5 over an index of the vault (updated on each write).

The **Obsidian** backend is identical to Markdown files but uses the user's actual Obsidian
vault path, so notes appear in Obsidian automatically.

---

## Translation Tool

```
text.translate
  → Translate text from one language to another
  Input: { "text": string, "to": string, "from"?: string }
    // "to" and "from": ISO 639-1 language codes ("en", "es", "fr", "de", "ja", etc.)
    // "from": auto-detected if omitted
  Output: { "translatedText": string, "detectedLanguage"?: string }
```

**Backends** (in priority order):

1. **On-device OPUS-MT** (privacy-safe, ~50MB per language pair, ONNX format):
   - High-quality open-source neural MT models from Helsinki-NLP
   - Downloaded on demand per language pair
   - Works in Privacy Mode
   - ~200ms per sentence on mid-range device

2. **LibreTranslate** (self-hosted, LAN-only option):
   - User can configure a self-hosted LibreTranslate instance URL
   - Works in LAN-only mode

3. **DeepL Free API** (cloud, opt-in, standard mode only):
   - Free tier: 500K chars/month
   - Requires DeepL API key stored in keystore
   - Blocked in Privacy Mode

**Language pair availability**: OPUS-MT covers 1,000+ language pairs. If the requested
pair is not available on-device, Karmik falls back to cloud (with user consent) or
returns an error with the closest available pair.

---

## New Built-in Agents (Productivity)

| Agent | Mode | Default Tools | Purpose |
|---|---|---|---|
| **Focus Coach** | ReAct | `focus.*`, `tasks.list`, `calendar.list_events` | Runs Pomodoro sessions; suggests tasks; delivers completion stats |
| **Daily Journal** | ReAct | `microphone.record`, `stt.transcribe`, `notes.create`, `memory.store` | Voice journaling → structured, tagged markdown note |
| **Daily Standup** | Autopilot | `tasks.list`, `calendar.list_events`, `git.log` | Drafts daily standup update; delivers as notification each morning |
| **Expense Tracker** | ReAct + Co-pilot | `screen.ocr`, `camera.capture_photo`, `memory.store`, `notes.append` | Scans receipts → structured expense entry in notes |
| **Wellness Check** | Autopilot | `health.*`, `tasks.list`, `weather.current` | Morning health summary: sleep, steps, today's weather + activity suggestion |
| **Reading Digest** | Autopilot | `readinglist.list`, `readinglist.read` | Daily summary of unread saved articles, grouped by topic |
