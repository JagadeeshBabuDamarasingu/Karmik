# Karmik — Model Context Protocol (MCP)

## Overview

Karmik supports MCP in both directions:

1. **Client mode**: Karmik connects to external MCP servers, discovers their tools, and exposes
   them to agents as first-class tools in the Tool Registry.
2. **Server mode**: Karmik runs its own MCP server (`localhost:5173` by default), allowing
   external MCP clients (Claude Code, VS Code Copilot, custom scripts) to call Karmik's tools
   and invoke Karmik agents.

This spec replaces the thin `McpToolProvider` stub in `specs/07-tool-registry.md`.

---

## MCP Client (Karmik Connects to External Servers)

### Transport Types

| Transport | Description | Platform |
|---|---|---|
| `stdio` | Spawn a local process; communicate over stdin/stdout | macOS, Windows, Linux only |
| `websocket` | Persistent WebSocket connection (`ws://` or `wss://`) | All platforms |
| `http_sse` | HTTP POST for requests, SSE stream for responses (MCP HTTP transport) | All platforms |

### Server Configuration

```dart
class McpServerConfig {
  final String id;                    // UUID, immutable
  final String displayName;           // shown in settings UI
  final McpTransport transport;       // stdio | websocket | http_sse
  final String endpoint;              // process command, ws:// URL, or http base URL
  final Map<String, String> env;      // environment vars (stdio only)
  final List<String> args;            // CLI args (stdio only)
  final String? authToken;            // Bearer token (stored in platform keystore)
  final bool autoStart;               // start server when Karmik launches (stdio only)
  final bool trustAllCerts;           // allow self-signed TLS (wss:// / https://)
  final Duration connectTimeout;      // default 10s
}

enum McpTransport { stdio, websocket, http_sse }
```

### Tool Discovery and Registration

On session start, Karmik calls `tools/list` on each configured and enabled MCP server.
Discovered tools are translated to Karmik's `Tool` interface and registered in the Tool
Registry for that session.

**Namespacing**: tool names are prefixed to avoid conflicts:

```
mcp.{serverId}.{toolName}

Examples:
  mcp.filesystem.read_file
  mcp.memory.store_memory
  mcp.postgres.query
```

**Tool schema translation**: MCP's JSON Schema input schemas are mapped to Karmik's
`ToolInput` type. JSON Schema features not supported by Karmik's validation layer
(e.g., `$ref`, `allOf`) are passed through as-is and validated by the MCP server.

### MCP Resources and Prompts

Karmik exposes two synthetic tools per MCP server so agents can consume all MCP
capabilities, not just tools:

```
mcp.{serverId}.read_resource
  Input: { "uri": string }
  Output: { "content": string, "mimeType": string }

mcp.{serverId}.use_prompt
  Input: { "name": string, "arguments"?: {} }
  Output: { "messages": [{ "role": string, "content": string }] }
```

### Connection Lifecycle

```
Session start
  │
  ├─ For each enabled McpServerConfig:
  │     ├─ stdio: spawn process, send initialize, receive serverInfo + capabilities
  │     ├─ websocket: open connection, send initialize
  │     └─ http_sse: POST /initialize, open SSE stream
  │
  ├─ Call tools/list → register tools as mcp.{id}.{name}
  ├─ Call resources/list (if supported) → register read_resource
  ├─ Call prompts/list (if supported) → register use_prompt
  │
Session running
  │
  ├─ Agent calls mcp.filesystem.read_file → Karmik sends tools/call to MCP server → return result
  │
Session end
  │
  └─ stdio: terminate child process; websocket: close; http_sse: close SSE stream
```

**Reconnection**: if the connection drops mid-session, Karmik attempts reconnect with
exponential backoff (1s, 2s, 4s, max 30s). Tool calls made during reconnection are queued
and retried up to 3 times.

**Stdio process management**: stdio MCP servers are managed as child processes of the
Karmik desktop app. Each `McpServerConfig` with `autoStart: true` has its process spawned
at app launch; others are spawned on first tool call and kept alive for the session.
Processes are killed when the session ends or the server is removed.

### Multi-Server Limits

- Maximum 10 MCP servers configured globally.
- Per-session: all enabled servers are connected; agents opt in to specific server
  namespaces via their `allowedTools` list (e.g., `["mcp.filesystem.*"]`).
- If a server fails to connect at session start, its tools are omitted for that session
  and a warning appears in the session info panel.

### Permission Model

```
tools.mcp.{serverId}.*   // grants all tools from one server
tools.mcp.filesystem.read_file  // grants one specific MCP tool
```

- Each `mcp.{serverId}.*` namespace requires explicit grant per agent (off by default).
- If "Approve new MCP tools on first use" is enabled in settings, the user sees an approval
  prompt the first time each new `mcp.*` tool is called, even within a granted namespace.
- MCP tool calls appear in the Co-pilot card like native tool calls, with the MCP server
  name shown as the source.

### Privacy Mode

| Transport | Privacy Mode behavior |
|---|---|
| `stdio` (local process) | Allowed — process runs locally |
| `websocket ws://` (LAN IP) | Allowed in LAN-only mode |
| `websocket wss://` (public) | Blocked |
| `http_sse http://` (LAN IP) | Allowed in LAN-only mode |
| `http_sse https://` (public) | Blocked |

MCP servers are checked against the same LAN IP detection as other network tools
(RFC 1918 ranges: 10.x, 172.16-31.x, 192.168.x).

---

## MCP Server (Karmik as an MCP Server)

Karmik exposes itself as an MCP server so external clients can use Karmik's tools and
invoke agents from any MCP-compatible tool.

### Server Details

- **Transport**: `http+sse` only (no stdio; Karmik is not a subprocess)
- **Bind address**: `localhost:{port}` (default `5173`, user-configurable)
- **Mobile**: bound to `0.0.0.0` to allow LAN access; NOT port-forwarded to internet
- **Desktop**: can optionally bind to a specific network interface (Settings → MCP → Server binding)
- **Lifecycle**: server starts when "Expose as MCP server" is enabled; runs as part of the
  background service (Android foreground service, macOS LaunchAgent, etc.)

### Authentication

- Optional Bearer token (auto-generated UUID v4, shown in Settings → MCP → Server token).
- If auth is enabled: all requests must include `Authorization: Bearer {token}`.
- Token can be rotated at any time; existing connections are dropped on rotation.
- Auth is disabled by default for localhost; recommended to enable when binding to 0.0.0.0.

### Exposed Capabilities

#### Tools

Each Karmik tool the user explicitly allows for MCP export is exposed with its native
name and schema. Enabled per tool in Settings → MCP → Exposed tools.

Default exposed (when MCP server is first enabled):
- `calendar.list_events`, `calendar.create_event`
- `tasks.list`, `tasks.create`
- `memory.store`, `memory.search`
- `contacts.search`

Never exposed (always blocked, even if user tries to toggle):
- `sms.send` (too destructive for remote callers)
- `git.push` (requires Co-pilot; not meaningful in remote context)
- `shell.run` (arbitrary command execution — too dangerous for remote callers)
- `devenv.env` (could leak secrets)

#### Agent Invocation

```
karmik.run_agent
  → Invoke a named Karmik agent with a task
  Input: { "agentId": string, "task": string, "waitForResult"?: bool }
  Output: { "sessionId": string, "result"?: string }
  Note: if waitForResult is false (default), returns immediately with sessionId;
        result is delivered asynchronously as a Karmik notification.
        if waitForResult is true (max 60s), blocks until the agent responds or times out.
```

#### Memory Search

```
karmik.memory_search
  → Semantic search over the user's Karmik memory
  Input: { "query": string, "limit"?: int, "agentId"?: string }
  Output: [{ "id", "content", "agentName", "createdAt": ISO8601, "score": float }]
  Note: searches shared memory pool; respects agent memory scope settings
```

#### Server Metadata

Karmik advertises itself with these MCP server capabilities:
```json
{
  "name": "karmik",
  "version": "1.0.0",
  "capabilities": {
    "tools": {},
    "resources": {},
    "prompts": {}
  }
}
```

### MCP Resources (Server mode)

Karmik exposes memory entries and snippets as MCP resources:

```
karmik://memory/{id}         → a single memory entry
karmik://snippets/{id}       → a code snippet
karmik://agents/{agentId}    → agent config summary (name, description, available tools)
```

External clients can call `resources/read` to fetch these directly without going through
the tool call flow.

### Privacy Mode

The MCP server itself is local (localhost binding) — it is always allowed in Privacy Mode.
However, any Karmik tool called *through* the MCP server is subject to the same Privacy
Mode restrictions as when called natively. An external client calling `browser.fetch` via
Karmik's MCP server will be blocked if Privacy Mode is on.

---

## MCP Management UI (Settings → MCP)

```
MCP Servers
══════════════════════════════════════════════════

CLIENT MODE  (Karmik connects to external servers)
──────────────────────────────────────────────────
  ● filesystem                                    [Edit] [✕]
    stdio · npx @modelcontextprotocol/server-filesystem ~/
    Status: connected · 12 tools available

  ○ memory                                        [Edit] [✕]
    websocket · ws://localhost:3001
    Status: disconnected (server not running)

  [+ Add MCP server]

SERVER MODE  (external tools connect to Karmik)
──────────────────────────────────────────────────
  ● Listening on localhost:5173          [Stop server]
  Token: karmik_f3a9…c72b               [Copy] [Rotate]

  Exposed tools:
    calendar.list_events        [✓]
    calendar.create_event       [✓]
    tasks.list                  [✓]
    tasks.create                [✓]
    memory.store                [✓]
    memory.search               [✓]
    contacts.search             [✓]
    smarthome.list_devices      [✗]
    files.read                  [✗]
    ...
    [Show all tools]

  [Configure exposed tools]
══════════════════════════════════════════════════
```

### Add MCP Server Dialog

```
Add MCP Server
──────────────────────────────
Display name:  [_______________]

Transport:     (●) stdio   ( ) WebSocket   ( ) HTTP+SSE

Command:       [npx @modelcontextprotocol/server-filesystem]
Arguments:     [~/]
Environment:   [KEY=VALUE  + Add]

Auth token:    [_______________ (optional)]
Auto-start:    [✓ Start when Karmik launches]

                               [Test connection]  [Save]
```

After "Test connection": Karmik connects, calls `tools/list`, and shows:
```
✓ Connected · 12 tools discovered:
  read_file, write_file, list_directory, create_directory, ...
```

---

## Example Flows

### Claude Code using Karmik's calendar

```
Claude Code (external MCP client)
  → POST http://localhost:5173/mcp  { method: "tools/call",
                                       params: { name: "calendar.list_events",
                                                 arguments: { "date": "2026-06-14" } } }
Karmik MCP server
  → validates Bearer token
  → checks calendar.list_events is in exposed tools list
  → calls native CalendarTool.run({ date: "2026-06-14" })
  → streams result back via SSE
Claude Code receives:
  → [{ title: "Standup", time: "09:00", ... }, ...]
```

### Karmik agent using a filesystem MCP server

```
User: "List all Python files in my project and find the one that imports numpy"
        │
Git Assistant (ReAct):
  → mcp.filesystem.list_directory { path: "~/project" }
    ✓ [list of files]
  → mcp.filesystem.read_file { path: "~/project/data_pipeline.py" }
    ✓ [file contents — contains "import numpy as np"]
  Response: "Found it: data_pipeline.py (line 3: import numpy as np)"
```

### Invoking a Karmik agent from VS Code

```javascript
// VS Code extension using Karmik as MCP server
const result = await mcpClient.callTool("karmik.run_agent", {
  agentId: "morning-briefing",
  task: "Give me a quick summary of today's tasks",
  waitForResult: true
});
// result.result = "You have 4 tasks today. 3 are due before noon..."
```

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| MCP client (websocket) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| MCP client (http_sse) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| MCP client (stdio) | — | — | ✓ | ✓ | ✓ | — |
| MCP server (expose Karmik) | ✓ | — | ✓ | ✓ | ✓ | — |
| Auto-start stdio servers | — | — | ✓ | ✓ | ✓ | — |

**iOS**: MCP server not exposed (no persistent background server on iOS; app is foreground-only).
**Web**: stdio not supported (no process spawning in browser); MCP server not available.
