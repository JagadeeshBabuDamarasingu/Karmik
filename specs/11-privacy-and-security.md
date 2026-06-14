# Karmik — Privacy & Security

## Core Privacy Principles

1. **Local by default** — all inference, memory, and tool execution happens on-device.
   No data leaves the phone unless the user explicitly enables a remote provider.
2. **No telemetry, no analytics** — Karmik collects nothing. No crash reporting services,
   no usage analytics, no model training opt-outs (there is no opt-in to begin with).
3. **Audit everything** — every action taken by every agent is logged before it executes.
   The user always knows what Karmik did and what data it accessed.
4. **User-controlled permissions** — no tool runs without explicit per-agent permission grants.
   Permissions can be revoked at any time.
5. **Plugin isolation** — plugins cannot access resources not declared in their manifest.

## Data Classification

| Data Type | Storage | Encrypted | Leaves Device |
|---|---|---|---|
| Conversation history | SQLite (on-device) | Yes (SQLCipher) | Never |
| Long-term memories | sqlite-vec (on-device) | Yes (SQLCipher) | Never |
| Model weights | App-private filesystem | No (large files) | Never |
| Plugin assets | App-private filesystem | No | Never |
| API keys | Android Keystore | Yes (hardware) | Only to provider |
| Audit log | SQLite (on-device) | Yes (SQLCipher) | Never |
| Task / todo data | SQLite (on-device) | Yes (SQLCipher) | Never |
| Notification content read by agents | Only in RAM during inference | — | Never |

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
