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

## MCP Server Support

Agents can connect to any MCP (Model Context Protocol) server as a tool source. MCP tools are
registered dynamically at session start.

```dart
class McpToolProvider {
  final String serverUrl;      // "ws://localhost:3000" or "stdio://..."
  final String? authToken;

  Future<List<Tool>> discoverTools();
}
```

MCP tool definitions are fetched from the server's `tools/list` endpoint and translated to the
Karmik `Tool` interface. Results are passed back to the model in the standard tool result format.

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
