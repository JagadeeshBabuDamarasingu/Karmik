# Karmik — Privacy & Security

## Core Privacy Principles

1. **Local by default** — all inference, memory, and tool execution happens on-device.
   No data leaves the device unless the user explicitly enables a remote provider.
2. **No telemetry, no analytics** — Karmik collects nothing. No crash reporting services,
   no usage analytics, no model training opt-outs (there is no opt-in to begin with).
3. **Audit everything** — every action taken by every agent is logged before it executes.
   The user always knows what Karmik did and what data it accessed.
4. **User-controlled permissions** — no tool runs without explicit per-agent permission grants.
   Permissions can be revoked at any time.
5. **Plugin isolation** — plugins cannot access resources not declared in their manifest.
6. **Privacy Mode** — a lockdown mode where the AI layer is guaranteed to make zero external
   network calls. Enforced at the network socket level, not just at the UI level.

## Data Classification

| Data Type | Storage | Encrypted | Leaves Device |
|---|---|---|---|
| Conversation history | SQLite (on-device) | Yes (SQLCipher) | Never |
| Long-term memories | sqlite-vec (on-device) | Yes (SQLCipher) | Never |
| Model weights | App-private filesystem | No (large files) | Never |
| Plugin assets | App-private filesystem | No | Never |
| API keys | Platform secure store¹ | Yes (hardware-backed) | Only to provider |
| Audit log | SQLite (on-device) | Yes (SQLCipher) | Never |
| Task / todo data | SQLite (on-device) | Yes (SQLCipher) | Never |
| Notification content read by agents | Only in RAM during inference | — | Never |
| Network activity log | SQLite (on-device) | Yes (SQLCipher) | Never |

¹ Platform secure store: Android Keystore, iOS/macOS Keychain, Windows Credential Manager,
Linux libsecret (Secret Service). See `specs/15-platform-support.md` for per-platform details.

## Encryption at Rest

All SQLite databases use **SQLCipher** (AES-256-CBC). The encryption key:
- Generated on first launch using `SecureRandom`
- Stored in the **Android Keystore** (hardware-backed on supported devices)
- Never written to disk in plaintext
- Never included in Android backups (databases excluded via `BackupAgent` configuration)

```dart
// Key derivation
final key = await secureKeyStore.getOrCreate('db_encryption_key');
final db = await openEncryptedDatabase('karmik.db', key: key);
```

## Encryption in Transit

When remote providers are used (user opt-in):
- All requests use TLS 1.3 minimum
- Certificate pinning for Anthropic and Google Gemini endpoints
- API keys transmitted only in Authorization headers (never in URL or body)
- No request caching by the OS HTTP stack (Cache-Control: no-store)

## Android Keystore Usage

```
Keys stored in Android Keystore:
  karmik_db_key         — AES-256 key for SQLite encryption
  karmik_vector_db_key  — AES-256 key for vector store encryption
  provider_{id}_apikey  — Encrypted API keys per provider
```

Keys are bound to the device (cannot be exported). If the device is factory-reset,
all keys are destroyed and data is unrecoverable (by design).

## Privacy Audit Log

Every agent tool invocation is logged *before* it executes. The log is append-only
(no modification, only deletion by the user).

### Schema

```sql
CREATE TABLE audit_log (
  id          TEXT PRIMARY KEY,
  agent_id    TEXT NOT NULL,
  agent_name  TEXT NOT NULL,
  session_id  TEXT NOT NULL,
  tool_id     TEXT NOT NULL,
  tool_name   TEXT NOT NULL,
  input_summary TEXT NOT NULL,   -- sanitized summary, never raw content
  outcome     TEXT NOT NULL,     -- "success" | "denied" | "error"
  created_at  INTEGER NOT NULL
);
```

`input_summary` is a sanitized description — it describes *what data was accessed*
without reproducing the full content. Example:
- `notifications.list` → "Read 3 notifications from WhatsApp and Gmail"
- `sms.send` → "Send SMS to +1-555-xxx-xxxx (8 chars)" (number partially masked)
- `http.get` → "GET https://api.example.com/..."

### Retention

Audit log is retained for 90 days by default. User can change the retention period or
clear the log entirely. Clearing the log does not affect conversation history or memories.

## Tool Permission System

```dart
class PermissionChecker {
  /// Returns true only if all three checks pass
  bool canExecute(String toolId, String agentId) {
    return _isToolAllowedForAgent(toolId, agentId)   // agent config check
        && _isAndroidPermissionGranted(toolId)        // OS permission check
        && !_isBlockedByActiveRule(toolId, agentId);  // plugin rule check
  }
}
```

Tools in the "dangerous" category (SMS send, file delete, contacts write) also require
that the agent is in Co-pilot mode OR the user has explicitly enabled "autonomous dangerous
actions" for that agent (off by default, with a stern warning).

## Plugin Security

Plugins are sandboxed via Dart isolates:

```
Plugin script isolate
  ✓ dart:core, dart:math, dart:convert
  ✓ Karmik Script API (declared tools only)
  ✗ dart:io (no filesystem access)
  ✗ dart:ffi (no native code)
  ✗ Platform channels (no Android API access)
  ✗ Network (unless tools.http.* declared in manifest)
```

Plugins cannot:
- Access other plugins' data
- Read Karmik internals (agent configs, memory, audit log, API keys)
- Modify the audit log
- Escalate their own permissions at runtime

Plugin scripts are reviewed (signature-verified) in the future marketplace. For now
(Stage 1, local install), users install plugins at their own risk — a clear warning is shown.

## Remote Inference Privacy Notice

When a user enables a remote provider:
- A clear one-time notice is shown: "Messages sent to [Provider Name] will be processed on
  their servers under their privacy policy."
- A persistent indicator is shown in the chat UI (colored dot, provider name) whenever
  a remote provider is active for a session
- Users can configure: "Always ask before using remote inference" (requires explicit
  confirmation per session)
- The audit log marks each inference turn with whether it was local or remote, and which provider

## Privacy Mode

Privacy Mode is a single global toggle (Settings → Privacy → Privacy Mode). When enabled,
it enforces that the AI layer — inference, memory operations, model catalog — makes zero
external network connections. The user's data stays on-device, unconditionally.

### What Privacy Mode blocks

| Component | Normal mode | Privacy Mode |
|---|---|---|
| Inference | Local or remote (user choice) | Local only — remote providers disabled |
| Model catalog refresh | Fetched from CDN periodically | Uses bundled/cached catalog only |
| LAN auto-discovery (Ollama) | Enabled | Disabled (local network scanning counts as external) |
| `http.get` / `browser.fetch` tools | Allowed per agent permissions | Blocked — return `success: false` with `"Privacy Mode is active"` |
| `http.post` tool | Allowed per agent permissions | Blocked |
| Plugin HTTP skills | Allowed if declared in manifest | Blocked |
| MCP servers (remote) | Allowed | Blocked |
| MCP servers (stdio/local) | Allowed | Allowed (local process, no network) |
| Embedding model download | Downloads from internet | Must already be downloaded |
| Background model downloads | Allowed | Blocked |

**What Privacy Mode does NOT block** (these are always local):
- All SQLite reads/writes
- Local inference (llama.cpp, ONNX, MLC-LLM)
- Memory operations (embed + retrieve)
- Calendar, contacts, SMS, notification tools (local system APIs)
- Plugin Dart scripts (sandboxed isolate, no network by default)

### Enforcement

Privacy Mode is not just a UI preference — it's enforced at the `ToolContext` level and
at the `ModelRuntimeFactory` level:

```dart
// In ToolContext:
if (privacyModeEnabled && tool.requiresNetwork) {
  return ToolResult(
    success: false,
    error: "Privacy Mode is active. Network tools are disabled. "
           "Disable Privacy Mode in Settings → Privacy to use this tool.",
    resultType: "text",
  );
}

// In ModelRuntimeFactory:
if (privacyModeEnabled && config.isRemote) {
  throw PrivacyModeViolation(
    "Cannot use remote provider '${config.displayName}' while Privacy Mode is active."
  );
}
```

A second enforcement layer uses the platform's network permission APIs to block outbound
connections from the app process entirely when Privacy Mode is on:
- **Android**: `NetworkSecurityConfig` with `<domain-config cleartextTrafficPermitted="false">`
  + a VPN-based network monitor that blocks all connections except an allowlist
- **macOS/iOS**: `NEContentFilter` network extension for app-level traffic blocking
- **Windows**: Windows Filtering Platform (WFP) callout driver — or simpler: block at the
  `HttpClient` layer by replacing the default client with a no-op stub
- **Linux**: `iptables` OUTPUT rule scoped to the Karmik process UID

This two-layer approach (app-layer + OS-layer) means even a buggy or malicious plugin cannot
exfiltrate data when Privacy Mode is on.

### Privacy Mode indicator

Privacy Mode shows a persistent status indicator throughout the UI:

```
┌────────────────────────────────────┐
│  🔒 Privacy Mode  ·  Local only    │  ← always-visible banner in main app
└────────────────────────────────────┘
```

In the overlay (bubble): the bubble changes from its normal color to deep blue when Privacy
Mode is active, providing a clear at-a-glance indicator even when using other apps.

### LAN-only mode (relaxed variant)

A softer option: "LAN-only mode" allows connections to private IP ranges (RFC 1918) only.
This enables:
- Ollama on a home server
- LM Studio on a local PC
- Self-hosted MCP servers

LAN-only is still stricter than the default (blocks all public internet) while being more
flexible than full Privacy Mode.

```
Privacy controls (Settings → Privacy):
  ● Full Privacy Mode    — no network at all from AI layer
  ○ LAN-only mode        — local network only (192.168.x.x, 10.x.x.x, 172.16.x.x)
  ○ Standard mode        — full network, per-agent permissions apply
```

---

## Expanded User Privacy Controls

### Network Activity Monitor

A live log of all outbound connections made by Karmik (when not in Privacy Mode). Accessible
from Settings → Privacy → Network Activity.

```
Network Activity
──────────────────────────────────────────────
Jun 14 · 07:02  Morning Briefing
  ● [BLOCKED by Privacy Mode]

Jun 13 · 22:15  Aria (manual)
  ✓ GET api.openweather.org:443        (http.get tool)
  ✓ GET example.com:443                (browser.fetch tool)

Jun 13 · 14:30  Provider: claude-sonnet-4-6
  ✓ POST api.anthropic.com:443         (inference)
  ✓ 1,203 tokens in / 412 tokens out  ·  $0.010
```

All entries include: agent name, destination hostname + port, purpose (inference / tool / plugin),
outcome (allowed / blocked / failed), and token/cost data for inference calls.

### Per-Agent Network Isolation

Beyond global Privacy Mode, each agent can be individually sandboxed:

```dart
class AgentConfig {
  // ... existing fields
  final NetworkPolicy networkPolicy;
}

enum NetworkPolicy {
  inherit,        // use global setting (default)
  localOnly,      // this agent: local inference + no network tools
  lanOnly,        // this agent: LAN-only
  unrestricted,   // this agent: full network (requires global to be standard mode)
}
```

UI: Agent Settings → Privacy → "Network access for this agent" picker.

### Notification Content Masking

When Karmik delivers result notifications, the content may include snippets of processed
data (task titles, calendar events, summarized messages). For users who don't want AI
results visible on their lock screen or in notification shade:

Settings → Privacy → "Mask notification content":
- **Off** (default): full result shown in notification
- **Summary only**: shows "Karmik: [Agent] completed a task" — no content
- **Silent**: delivers result to in-app inbox only; no OS notification

### Data Retention Controls

Fine-grained per-agent data lifecycle settings (extends the existing `historyRetentionDays`):

```
Agent: Aria — Data retention
  Conversation history:  [ 30 days ▼ ]
  Long-term memories:    [ Keep forever ▼ ]  [Delete all memories]
  Audit log entries:     [ 90 days ▼ ]
  Auto-summarize to memory at session end:  [✓]
  Export before delete:  [✓]
```

### Selective Memory Deletion

From the Memory Browser, the user can:
- Delete individual memories by tapping and holding
- Filter by date range, agent, or tag → bulk-delete
- "Forget everything before [date]" — purge memories older than a chosen date
- Export selected memories as JSON before deleting

### Third-Party Share Confirmation

When another app shares content into Karmik (via the Share Sheet, `specs/10-ui.md` Surface 4),
a privacy notice is shown before the content is processed:

```
┌────────────────────────────────────────┐
│ Content shared from Chrome             │
│                                        │
│ This text will be processed by:        │
│  Agent: Aria                           │
│  Model: Gemma 4 2B (on-device) 🔒     │
│                                        │
│ No data will leave your device.        │
│                                        │
│ [Continue]            [Cancel]         │
└────────────────────────────────────────┘
```

If a remote model is active, the notice instead reads: "This content will be sent to
[Provider Name] for processing." The user must explicitly confirm each time a new content
type or source app is shared, until they tick "Don't ask again for Chrome."

## Data Export & Deletion

**Export** (Settings → Privacy → Export my data):
- Conversations: JSON export of all message history per agent
- Memories: JSON export of all long-term memories
- Tasks: JSON export of all tasks
- Audit log: JSON export

**Deletion** (Settings → Privacy):
- Delete all data for a specific agent
- Delete all data (factory reset Karmik): clears all databases, revokes all Keystore keys,
  unregisters background service triggers
- Individual memory deletion via the Memory Browser
- Individual audit log entries cannot be deleted individually (only bulk clear)

## No Cloud Sync (Stage 1)

Karmik does not sync any data to any cloud service in Stage 1. There is no Karmik account.
There is no server. There is no company backend. The app is fully self-contained.

This is both a privacy feature and a simplicity feature. Stage 2 may introduce
optional encrypted cross-device sync (user-controlled, using the user's own cloud storage
such as Google Drive or a self-hosted server), but this will be strictly opt-in.
