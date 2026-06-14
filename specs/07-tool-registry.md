# Karmik — Tool Registry

## Overview

The Tool Registry is the central catalog of everything an agent can *do*. Every capability —
reading a notification, making an HTTP request, recording audio — is a registered tool with a
consistent interface.

Tools are declared in a standard JSON schema format compatible with OpenAI function calling and
Anthropic tool use. This means the same tool definitions work across all inference backends.

## Tool Interface

```dart
abstract class Tool {
  String get id;
  String get name;
  String get description;
  Map<String, dynamic> get inputSchema;  // JSON Schema

  Future<ToolResult> execute(Map<String, dynamic> input, ToolContext context);
}

class ToolContext {
  final String agentId;
  final String sessionId;
  final AuditLogger auditLogger;     // every execution is logged before it runs
  final PermissionChecker permissions;
}

class ToolResult {
  final bool success;
  final dynamic data;                // text, JSON, file path, or base64 image
  final String? error;
  final String resultType;           // "text" | "json" | "file" | "image"
}
```

## Permission Model

Before any tool executes, `PermissionChecker` verifies:
1. The tool is in the agent's `allowedTools` list
2. The required Android permission is granted
3. The tool is not blocked by an active rule from a plugin

Tools never execute without passing all three checks. Failures return a `ToolResult` with
`success: false` and a descriptive error the agent can reason about.

## Built-in Tool Catalog

### Notification Tools

```
notifications.list
  → List recent notifications, optionally filtered by app or time range
  Input: { "app": string?, "since": ISO8601?, "limit": int? }
  Output: [{ "id", "app", "title", "body", "timestamp", "actions": [] }]

notifications.dismiss
  → Dismiss a notification by ID
  Input: { "id": string }
  Output: { "dismissed": bool }

notifications.reply
  → Reply to a notification that supports inline reply (e.g. WhatsApp, SMS)
  Input: { "id": string, "reply": string }
  Output: { "sent": bool }
```

### Calendar Tools

```
calendar.list_events
  → List calendar events in a date range
  Input: { "start": ISO8601, "end": ISO8601, "calendarId": string? }
  Output: [{ "id", "title", "start", "end", "location", "attendees", "description" }]

calendar.create_event
  → Create a new calendar event
  Input: { "title", "start", "end", "description"?, "location"?, "attendees"?: [] }
  Output: { "id": string, "created": bool }

calendar.update_event
  → Update an existing event
  Input: { "id", "title"?, "start"?, "end"?, "description"?, "location"? }
  Output: { "updated": bool }

calendar.delete_event
  → Delete an event (requires Co-pilot confirmation by default)
  Input: { "id": string }
  Output: { "deleted": bool }

calendar.find_free_slots
  → Find free time slots in a date range across all calendars
  Input: { "start": ISO8601, "end": ISO8601, "durationMinutes": int }
  Output: [{ "start": ISO8601, "end": ISO8601 }]
```

### Task / Todo Tools

```
tasks.list
  → List tasks, optionally filtered by status, agent, or label
  Input: { "status"?: "open|done|all", "agentId"?: string, "limit"?: int }
  Output: [{ "id", "title", "notes", "status", "dueDate", "source", "agentId" }]

tasks.create
  → Create a task
  Input: { "title": string, "notes"?: string, "dueDate"?: ISO8601, "labels"?: [] }
  Output: { "id": string }

tasks.update
  → Update a task (status, title, due date)
  Input: { "id", "title"?, "status"?, "dueDate"?, "notes"? }
  Output: { "updated": bool }

tasks.delete
  → Delete a task
  Input: { "id": string }
  Output: { "deleted": bool }
```

### SMS / Messaging Tools

```
sms.list
  → List recent SMS conversations
  Input: { "contact"?: string, "limit"?: int }
  Output: [{ "contact", "messages": [{ "body", "timestamp", "direction" }] }]

sms.send
  → Send an SMS (always requires Co-pilot confirmation)
  Input: { "to": string, "body": string }
  Output: { "sent": bool, "messageId": string }
```

### File System Tools

```
files.list
  → List files in a directory
  Input: { "path": string, "recursive"?: bool, "filter"?: string }
  Output: [{ "name", "path", "size", "modifiedAt", "mimeType" }]

files.read
  → Read file content (text files; images returned as base64)
  Input: { "path": string }
  Output: { "content": string, "mimeType": string }

files.write
  → Write content to a file (creates or overwrites)
  Input: { "path": string, "content": string }
  Output: { "written": bool, "path": string }

files.delete
  → Delete a file (requires Co-pilot confirmation)
  Input: { "path": string }
  Output: { "deleted": bool }
```

### Media Tools

```
camera.capture_photo
  → Take a photo with the device camera
  Input: { "facing"?: "front|back" }
  Output: { "imagePath": string, "base64"?: string }

microphone.record
  → Record audio for a specified duration
  Input: { "durationSeconds": int, "transcribe"?: bool }
  Output: { "audioPath": string, "transcript"?: string }

stt.transcribe
  → Transcribe an audio file using Whisper.cpp (on-device)
  Input: { "audioPath": string, "language"?: string }
  Output: { "transcript": string, "language": string }
```

### Contact Tools

```
contacts.search
  → Search contacts by name or phone number
  Input: { "query": string }
  Output: [{ "name", "phone"?: string, "email"?: string }]

contacts.get
  → Get a contact by ID
  Input: { "id": string }
  Output: { "name", "phone"?: [], "email"?: [], "address"?: [] }
```

### Location Tools

```
location.current
  → Get current device location
  Input: { "accuracy"?: "low|medium|high" }
  Output: { "latitude": float, "longitude": float, "accuracy": float, "address"?: string }
```

### Smart Home Tools

Controls smart devices via the active `SmartHomeAdapter` (see `specs/16-smart-home.md`).
All tools operate against whichever hub(s) the user has configured. If no hub is configured,
every tool returns `success: false` with `"No smart home hub configured. Add one in Settings → Smart Home."`.

```
smarthome.list_devices
  → List all known smart devices and their current state
  Input: { "room"?: string, "type"?: string, "onlyOn"?: bool }
  Output: [{ "id", "name", "type", "room", "on": bool, "state": {} }]

smarthome.get_device
  → Get a specific device by ID or name
  Input: { "id"?: string, "name"?: string }
  Output: { "id", "name", "type", "room", "on": bool, "state": {}, "capabilities": [] }

smarthome.control
  → Send a command to a device or group of devices
  Input: {
    "id"?: string,           // device ID (preferred)
    "name"?: string,         // device name (fuzzy matched if ID unknown)
    "room"?: string,         // target all devices in a room
    "type"?: string,         // target all devices of a type (e.g. "light")
    "command": string,       // "on" | "off" | "toggle" | "set"
    "value"?: any            // for "set": number (brightness 0–100), string (color), object (thermostat settings)
  }
  Output: { "success": bool, "devicesAffected": int, "newState": {} }

smarthome.list_rooms
  → List all rooms / areas and a summary of their device states
  Input: {}
  Output: [{ "id", "name", "deviceCount": int, "devicesOn": int }]

smarthome.run_scene
  → Activate a saved scene or routine (e.g. "Good morning", "Movie time")
  Input: { "name": string }
  Output: { "success": bool, "scene": string }

smarthome.query_sensor
  → Read the current value from a sensor (temperature, humidity, motion, door, etc.)
  Input: { "id"?: string, "name"?: string, "type"?: string }
  Output: { "id", "name", "type", "value": any, "unit"?: string, "lastUpdated": ISO8601 }
```

**Permission**: `tools.smarthome.*` must be explicitly granted per agent. Default: off.
Smart home tools are **always local-network** — they never route through the internet unless
the configured hub uses a cloud API (Tuya, SmartThings). In Privacy Mode, cloud-hub adapters
are blocked; local-hub adapters (Home Assistant, Matter, Philips Hue LAN) are allowed.

### Network Tools

```
http.get
  → Make an HTTP GET request
  Input: { "url": string, "headers"?: {}, "timeoutSeconds"?: int }
  Output: { "status": int, "body": string, "headers": {} }

http.post
  → Make an HTTP POST request
  Input: { "url": string, "body": any, "headers"?: {}, "timeoutSeconds"?: int }
  Output: { "status": int, "body": string, "headers": {} }

browser.fetch
  → Fetch a webpage and return its text content (strips HTML)
  Input: { "url": string }
  Output: { "title": string, "content": string, "url": string }
```

### Memory Tools

```
memory.store
  → Store a piece of information in long-term semantic memory
  Input: { "content": string, "tags"?: [] }
  Output: { "id": string }

memory.search
  → Semantically search long-term memory
  Input: { "query": string, "limit"?: int, "tags"?: [] }
  Output: [{ "id", "content", "score": float, "createdAt" }]

memory.forget
  → Delete a memory entry
  Input: { "id": string }
  Output: { "deleted": bool }
```

### Prompt Tools

```
prompts.use
  → Fill a prompt template and return the result
  Input: { "promptId": string, "variables": {} }
  Output: { "prompt": string }
```

### Code Execution Tools

Sandboxed code execution in an isolated Dart isolate. No network, no filesystem access beyond
a temp directory. Designed for data transformation, quick calculations, and script-driven
reasoning — not for running arbitrary system commands.

```
code.run
  → Execute a Dart script in a sandboxed isolate and return its output
  Input: {
    "code": string,           // full Dart program with a main() entry point
    "timeoutSeconds": int?    // default 10, max 30
  }
  Output: { "stdout": string, "stderr": string, "exitCode": int }
  Sandbox: dart:core, dart:math, dart:convert only — no dart:io, no dart:ffi, no platform channels

code.eval_js
  → Evaluate a JavaScript expression using the QuickJS engine (bundled, no V8)
  Input: {
    "code": string,           // JS expression or function body
    "timeoutSeconds": int?    // default 5, max 15
  }
  Output: { "result": string, "error": string? }
  Sandbox: ES2020 core — no fetch, no DOM, no Node APIs
```

**Use cases**: unit conversions, date arithmetic, JSON transformation, markdown table
generation, regex extraction from text. For heavier computation, agents should use
`http.post` to call an external service instead.

**Permission**: `tools.code.run` must be explicitly granted per agent. Default: off.

### Karmik Internal Tools

```
agents.spawn
  → Spawn a sub-agent (Plan-and-Execute mode only)
  Input: { "agentId": string, "task": string, "context"?: string }
  Output: { "sessionId": string, "result": string }

karmik.notify
  → Send a notification to the user from a background agent
  Input: { "title": string, "body": string, "actions"?: [{ "label", "intent" }] }
  Output: { "delivered": bool }

karmik.checkpoint
  → Save a named checkpoint of the current session state (for rollback)
  Input: { "label": string? }
  Output: { "checkpointId": string }

karmik.rollback
  → Restore session state to a named checkpoint (removes messages after the checkpoint)
  Input: { "checkpointId": string }
  Output: { "restored": bool, "messagesRemoved": int }
```

### Email Tools

See `specs/17-productivity-tools.md` for full integration details (platform backends, Privacy Mode).

```
email.list
  → List emails from inbox or a folder
  Input: { "folder"?: string, "since"?: ISO8601, "limit"?: int, "unreadOnly"?: bool }
  Output: [{ "id", "from", "to": [], "subject", "date": ISO8601, "snippet", "read": bool }]

email.read
  → Read a single email with full body
  Input: { "id": string }
  Output: { "id", "from", "to": [], "cc": [], "subject", "body", "date": ISO8601,
            "attachments": [{ "name", "mimeType", "sizeKb" }] }

email.search
  → Search emails by keyword, sender, or date range
  Input: { "query": string, "limit"?: int }
  Output: [{ "id", "from", "subject", "date": ISO8601, "snippet" }]

email.compose
  → Draft an email (does not send; returns draft for Co-pilot review)
  Input: { "to": string[], "subject": string, "body": string, "cc"?: string[] }
  Output: { "draftId": string, "preview": string }

email.send
  → Send an email (Co-pilot required)
  Input: { "to": string[], "subject": string, "body": string, "cc"?: string[] }
  Output: { "sent": bool, "messageId": string }
  Co-pilot: always

email.reply
  → Reply to an email thread (Co-pilot required)
  Input: { "id": string, "body": string, "replyAll"?: bool }
  Output: { "sent": bool }
  Co-pilot: always

email.archive
  → Archive an email
  Input: { "id": string }
  Output: { "archived": bool }

email.label
  → Add or remove a label/folder on an email
  Input: { "id": string, "add"?: string[], "remove"?: string[] }
  Output: { "updated": bool }
```

**Permission**: `tools.email.*` off by default. `email.send` and `email.reply` require
separate explicit grant and always invoke Co-pilot.

### Clipboard Tools

```
clipboard.read
  → Read the current clipboard content
  Input: {}
  Output: { "text"?: string, "imagePath"?: string, "mimeType": string }

clipboard.write
  → Write text to the clipboard
  Input: { "text": string }
  Output: { "written": bool }
```

**Platform**: Android `ClipboardManager`; iOS `UIPasteboard`; macOS/Windows/Linux system clipboard.
**Privacy Mode**: clipboard is local-only — allowed in Privacy Mode.

### Screen Capture & OCR Tools

```
screen.capture
  → Take a screenshot of the current screen or a region
  Input: { "region"?: { "x": int, "y": int, "width": int, "height": int } }
  Output: { "imagePath": string }
  Platform: Android (MediaProjection API, requires one-time user grant);
            macOS (CGWindowListCreateImage); Windows (BitBlt); Linux (scrot/grim).
            Not available on iOS (sandbox) or Web.

screen.ocr
  → Extract text from an image or the live screen
  Input: { "imagePath"?: string }   // omit to capture + OCR current screen
  Output: { "text": string, "blocks": [{ "text", "boundingBox": {} }] }
  Implementation: Tesseract.cpp (on-device, ~30MB); falls back to Android ML Kit
                  or Apple VisionKit on those platforms.
```

**Permission**: `tools.screen.capture` off by default. Android requires `MEDIA_PROJECTION`
grant via a system dialog on first use.

### Focus Mode Tools

```
focus.start
  → Start a focus/Pomodoro session and enable Do-Not-Disturb
  Input: { "durationMinutes": int, "label"?: string }
  Output: { "sessionId": string, "endsAt": ISO8601 }

focus.status
  → Get the current focus session state
  Input: {}
  Output: { "active": bool, "label"?: string, "minutesRemaining"?: int,
            "completedCount": int }

focus.end
  → End the current focus session early
  Input: {}
  Output: { "ended": bool, "minutesCompleted": int }
```

**DND implementation**: Android `NotificationManager.setInterruptionFilter`; iOS Focus
framework; macOS Focus Modes; Windows Focus Assist (WinRT); Linux PulseAudio/PipeWire mute.

### Weather Tools

```
weather.current
  → Current weather conditions at the device's location (or a specified location)
  Input: { "latitude"?: float, "longitude"?: float }
  Output: { "temp": float, "feelsLike": float, "humidity": int, "wind": float,
            "windDir": string, "uvIndex": float, "condition": string, "icon": string }

weather.forecast
  → Multi-day forecast with hourly breakdown
  Input: { "latitude"?: float, "longitude"?: float, "days"?: int }
  Output: [{ "date": ISO8601, "high": float, "low": float, "condition": string,
             "hourly": [{ "time": ISO8601, "temp": float, "condition": string }] }]
```

**Provider**: Open-Meteo API (free, no API key, no account, sends GPS coordinates only).
Local cache TTL: 1 hour. **Privacy Mode**: Open-Meteo is public internet — blocked in
Privacy Mode (tool returns an error; user can enable a local weather server override).

### Health & Fitness Tools

```
health.steps
  → Step count for a date range
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: [{ "date": ISO8601, "steps": int }]

health.sleep
  → Sleep data for recent nights
  Input: { "nights"?: int }
  Output: [{ "date": ISO8601, "durationMinutes": int, "quality"?: "poor|fair|good|great" }]

health.activity
  → Workouts and active minutes for a period
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: [{ "date": ISO8601, "activeMinutes": int, "calories": int,
             "workouts": [{ "type", "durationMinutes", "calories" }] }]

health.heart_rate
  → Resting and peak heart rate readings
  Input: { "start": ISO8601, "end": ISO8601 }
  Output: [{ "timestamp": ISO8601, "bpm": int, "type": "resting|active|peak" }]
```

**Platform**: Android Health Connect (`androidx.health.connect`); iOS Apple HealthKit
(`HKHealthStore`). Not available on desktop or Web. **Privacy Mode**: all health data is
local-only — allowed in Privacy Mode.

### Reading List Tools

```
readinglist.add
  → Save a URL to the reading list with an AI-generated summary
  Input: { "url": string, "tags"?: string[] }
  Output: { "id": string, "title": string, "summary": string }

readinglist.list
  → List saved reading list items
  Input: { "unreadOnly"?: bool, "tag"?: string, "limit"?: int }
  Output: [{ "id", "title", "url", "summary", "tags": [], "addedAt": ISO8601,
             "read": bool }]

readinglist.read
  → Fetch the full article text of a saved item
  Input: { "id": string }
  Output: { "title": string, "content": string, "url": string }

readinglist.delete
  → Remove an item from the reading list
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: `readinglist` table in `karmik.db` with sqlite-vec embeddings for semantic
search. Full article text stored if < 100 KB. **Privacy Mode**: `readinglist.add` calls
`browser.fetch` (internet) — blocked. `readinglist.list` and `readinglist.read` (if cached)
are local-only — allowed.

### Note-Taking Tools

```
notes.create
  → Create a new note in the configured notes backend
  Input: { "title": string, "content": string, "tags"?: string[] }
  Output: { "id": string, "url"?: string }

notes.read
  → Read a note by ID or title
  Input: { "id"?: string, "title"?: string }
  Output: { "id", "title", "content", "tags": [], "updatedAt": ISO8601 }

notes.list
  → List recent notes
  Input: { "limit"?: int, "tag"?: string }
  Output: [{ "id", "title", "snippet", "tags": [], "updatedAt": ISO8601 }]

notes.search
  → Search notes by content (full-text + semantic)
  Input: { "query": string, "limit"?: int }
  Output: [{ "id", "title", "snippet", "score": float }]

notes.append
  → Append text to an existing note
  Input: { "id": string, "content": string }
  Output: { "updated": bool }

notes.update
  → Update a note's title, content, or tags
  Input: { "id": string, "title"?: string, "content"?: string, "tags"?: string[] }
  Output: { "updated": bool }
```

**Backends** (user picks in Settings → Notes):
- **Markdown files** (default): `~/Documents/karmik-notes/`; uses `files.*` internally
- **Obsidian**: local vault path via `files.*`; local-only
- **Apple Notes** (iOS/macOS): `CNNoteStore` / `ASFilesystemNote`
- **Notion** (cloud): Notion API; blocked in Privacy Mode

### Translation Tool

```
text.translate
  → Translate text between any supported languages
  Input: { "text": string, "to": string, "from"?: string }
    // "to" and "from": BCP-47 language codes (e.g. "es", "de", "zh")
  Output: { "translation": string, "detectedLanguage"?: string }
```

**Implementation**: on-device OPUS-MT ONNX (~50 MB per language pair, Helsinki-NLP);
falls back to LibreTranslate (self-hosted; configure URL in Settings → Translation) or
DeepL free tier. **Privacy Mode**: OPUS-MT on-device is allowed; cloud APIs are blocked.

### Git Tools

See `specs/18-developer-tools.md` for full details.

```
git.status    Input: { "cwd": string }
git.log       Input: { "cwd": string, "limit"?: int, "branch"?: string, "since"?: ISO8601 }
git.diff      Input: { "cwd": string, "staged"?: bool, "from"?: string, "to"?: string, "path"?: string }
git.blame     Input: { "cwd": string, "path": string, "lines"?: { "start": int, "end": int } }
git.branch    Input: { "cwd": string, "all"?: bool }
git.stash_list Input: { "cwd": string }
git.commit_message  Input: { "cwd": string, "style"?: "conventional|short|detailed" }
git.commit    Input: { "cwd": string, "message": string, "stageAll"?: bool }  // Co-pilot
git.push      Input: { "cwd": string, "remote"?: string, "branch"?: string, "force"?: bool }  // Co-pilot
```

**Platform**: macOS, Windows, Linux, Android (Termux). Not available on iOS or Web.
**Permission**: `tools.git.*` off by default. Read tools granted as a group; write tools
(`git.commit`, `git.push`) require separate explicit grant.

### GitHub / GitLab Tools

See `specs/18-developer-tools.md` for full details. GitLab equivalents use the `gitlab.*`
namespace. Bitbucket basic read support under `bitbucket.*`.

```
github.list_issues   Input: { "repo"?: string, "state"?: "open|closed|all", "label"?: string }
github.get_issue     Input: { "repo"?: string, "number": int }
github.create_issue  Input: { "repo"?: string, "title": string, "body"?: string }
github.list_prs      Input: { "repo"?: string, "state"?: "open|closed|all" }
github.get_pr        Input: { "repo"?: string, "number": int }
github.comment       Input: { "repo"?: string, "number": int, "body": string }  // Co-pilot
github.create_pr     Input: { "repo"?: string, "title": string, "head": string, "base"?: string }  // Co-pilot
```

**Permission**: `tools.github.*` off by default; PAT stored in platform keystore.
**Privacy Mode**: GitHub/GitLab APIs are public internet — blocked. Self-hosted GitLab on
LAN is allowed in LAN-only mode.

### Shell Tool

```
shell.run
  → Execute a shell command in a sandboxed subprocess
  Input: {
    "command": string,
    "cwd"?: string,          // defaults to user's home directory
    "timeoutSeconds"?: int,  // default 30, max 300
    "env"?: {}
  }
  Output: { "stdout": string, "stderr": string, "exitCode": int }
  Platform: macOS (zsh), Linux (bash), Windows (PowerShell), Android/Termux (bash).
            NOT available on iOS or Web.
```

Blocked patterns (checked via regex): `rm -rf /`, `rm -rf ~`, `sudo`, `su`, `doas`, `dd`,
`mkfs`, `fdisk`, `parted`, `shred`, `shutdown`, `reboot`, `halt`, `poweroff`, `chmod 777 /`.
Working directory must be within user's home or a configured project root.

**Permission**: `tools.shell.run` off by default. Requires explicit grant with a warning
about arbitrary command execution risk.

### Dev Environment Tools

```
devenv.ports      → List open listening ports  Input: { "filter"?: string }
devenv.processes  → List running processes     Input: { "name"?: string, "limit"?: int }
devenv.docker     → List Docker containers     Input: {}
devenv.env        → Read env variables (allowlist) Input: { "keys": string[] }
devenv.ping       → Check host/port reachability  Input: { "host": string, "port"?: int }
```

Implemented via `shell.run` internally. Not available on iOS or Web.
**Privacy Mode**: all `devenv.*` tools are local-only — allowed in Privacy Mode.
`devenv.env` has a denylist of secret variable names (`GITHUB_TOKEN`, `AWS_SECRET_KEY`, etc.)
that are always blocked regardless of the `keys` input.

### Documentation Browser Tools

```
docs.fetch
  → Download and index documentation from a URL or package name
  Input: { "url"?: string, "package"?: string, "version"?: string, "force"?: bool }
  Output: { "id": string, "title": string, "chunksIndexed": int, "sizeKb": int }

docs.search
  → Semantic search across all cached documentation
  Input: { "query": string, "source"?: string, "limit"?: int }
  Output: [{ "id", "source", "title", "chunk": string, "score": float, "url": string }]

docs.list
  → List all cached documentation sources
  Input: {}
  Output: [{ "id", "title", "url", "version"?: string, "chunksIndexed": int,
             "fetchedAt": ISO8601 }]

docs.delete
  → Remove a cached documentation source
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: `karmik_docs.db` (separate from `karmik.db`). Chunks at ~500 tokens with
50-token overlap; embedded with all-MiniLM-L6-v2. **Privacy Mode**: `docs.fetch` calls
`browser.fetch` — blocked. `docs.search` on cached data — allowed.

### Code Snippet Library Tools

```
snippets.store
  → Save a code snippet with description and tags
  Input: { "code": string, "language"?: string, "description": string, "tags"?: string[] }
  Output: { "id": string }

snippets.search
  → Semantic and full-text search over saved snippets
  Input: { "query": string, "language"?: string, "limit"?: int }
  Output: [{ "id", "description", "language", "tags": [], "code": string,
             "score": float, "createdAt": ISO8601 }]

snippets.get
  → Get a snippet by ID
  Input: { "id": string }
  Output: { "id", "description", "language", "tags": [], "code": string }

snippets.update
  → Update a snippet's metadata or code
  Input: { "id": string, "code"?: string, "description"?: string, "tags"?: string[] }
  Output: { "updated": bool }

snippets.delete
  → Delete a snippet
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: `snippets` table in `karmik.db`. Embeddings from `description + "\n\n" + code`.
FTS5-indexed for keyword search. **Privacy**: entirely local — allowed in Privacy Mode.

### TTS Tools

```
tts.speak
  → Speak text aloud using text-to-speech
  Input: { "text": string, "voice"?: string, "speed"?: float }
  Output: { "spoken": bool, "durationMs": int }
  Note: non-blocking; starts playback and returns. Use tts.stop to interrupt.

tts.stop
  → Stop current TTS playback
  Input: {}
  Output: { "stopped": bool }
```

**Privacy Mode**: Piper TTS (on-device) and platform TTS are allowed. Cloud TTS providers
are blocked. See `specs/20-voice.md` for full TTS documentation.

### Voice Tools

```
voice.mode_on
  → Activate voice-first hands-free mode (Co-pilot required)
  Input: { "reason"?: string }
  Output: { "active": bool }
  Co-pilot: always

voice.mode_off
  → Deactivate voice-first mode
  Input: {}
  Output: { "active": bool }
```

See `specs/20-voice.md` for voice-first mode behavior and UI.

### Computer Use Tools

See `specs/23-computer-use.md` for full implementation details (platform APIs, coordinate
system, element finder strategies, safety model).

```
computer.screenshot  → Screenshot of screen or region       Input: { "region"?: {x,y,w,h}, "display"?: int }
computer.find_element → Find UI element by text/image/role  Input: { "text"?: string, "role"?: string, "image"?: string }
computer.click       → Click at coordinate or element       Input: { "x"?: int, "y"?: int, "button"?: string, "clicks"?: int }  // Co-pilot
computer.type        → Type text at cursor                  Input: { "text": string, "clearFirst"?: bool }  // Co-pilot
computer.key         → Press keyboard shortcut              Input: { "key": string }  // Co-pilot, e.g. "cmd+c"
computer.scroll      → Scroll in a direction                Input: { "x"?: int, "y"?: int, "direction": string, "amount"?: int }
computer.drag        → Drag between coordinates             Input: { "fromX": int, "fromY": int, "toX": int, "toY": int }  // Co-pilot
computer.get_clipboard → Read clipboard                     Input: {}
computer.set_clipboard → Write clipboard text               Input: { "text": string }
```

```
automation.record      → Start recording a macro            Input: { "name": string }  // Co-pilot
automation.stop_record → Stop and save a macro              Input: { "recordingId": string }
automation.play        → Replay a saved macro               Input: { "macroId": string, "batchApproval"?: bool }  // Co-pilot
automation.list_macros → List saved macros                  Input: {}
automation.delete_macro → Delete a macro                    Input: { "id": string }
```

**Platform**: macOS, Windows, Linux only. Not available on Android, iOS, or Web.
**Permission**: `tools.computer.*` off by default. Requires explicit grant + Accessibility
permission (macOS/Linux) or no extra OS permission (Windows).
All input actions (`click`, `type`, `key`, `drag`) are always Co-pilot.

### Webhook Tool

```
webhook.send
  → POST JSON data to an external URL
  Input: { "url": string, "payload": {}, "headers"?: {}, "method"?: "POST|PUT|PATCH" }
  Output: { "status": int, "body": string }
  Co-pilot: first use per URL (user approves the destination domain)
```

**Permission**: `tools.webhook.send` off by default.
**Privacy Mode**: blocked for public internet URLs; allowed for LAN IPs.
See `specs/24-automation-workflows.md` for inbound webhooks and iOS Shortcuts integration.

Full MCP documentation (client and server) is in `specs/19-mcp.md`.

**Summary**:
- Karmik connects to external MCP servers (stdio, websocket, http+sse transports).
  Discovered tools are namespaced `mcp.{serverId}.{toolName}` and available to agents.
- Karmik exposes itself as an MCP server on `localhost:5173`, allowing external MCP
  clients (Claude Code, VS Code Copilot, etc.) to call Karmik's tools and invoke agents.
- Up to 10 MCP servers can be configured. Each requires explicit per-agent permission grant.
- Privacy Mode: local/LAN MCP servers allowed; public internet MCP servers blocked.

## Tool Permission Matrix

Default permissions per built-in agent type (user can override):

| Tool | Task Capture | Morning Briefing | Meeting Notes | Research |
|---|---|---|---|---|
| notifications.* | ✓ | ✓ | — | — |
| calendar.* | read only | read only | read+create | — |
| tasks.* | create | read | create | — |
| sms.* | — | — | — | — |
| files.* | — | — | write | write |
| camera.* | — | — | — | — |
| microphone.record | — | — | ✓ | — |
| http.* | — | — | — | ✓ |
| browser.fetch | — | — | — | ✓ |
| memory.* | ✓ | read | ✓ | ✓ |

Default for custom agents: **no tools**. User explicitly grants each tool category.
