# Karmik — Error Handling & Resilience

## Principles

1. **Agents see errors as context, not crashes.** Tool failures return a `ToolResult` with
   `success: false`; the agent receives this in its context window and can reason about it
   (retry, ask the user, or choose an alternative approach). Errors are data, not exceptions.
2. **Users see actions, not stack traces.** Every user-visible error message states: what
   happened, why it matters, and what the user can do.
3. **Fail safe over fail loud.** Prefer degraded but working (e.g., read-only mode, fallback
   model) over complete failure.
4. **Log everything, surface selectively.** All errors go to the audit log. Only errors
   requiring user action surface as UI notifications.

---

## Inference Errors

### Timeout (30-second wall clock limit)

The `ModelRuntime.infer()` stream has a 30-second timeout on first-token arrival.
If no token is received within 30s:

| Runtime type | Behavior |
|---|---|
| Local (llama.cpp) | Cancel via `ModelRuntime.cancel()`, free KV cache, surface: "Response timed out. The model may be overloaded. Try a shorter message or switch to a faster model." |
| Remote provider | Cancel HTTP request, surface: "No response from [Provider] after 30 seconds. Check your connection or switch to on-device inference." |

The user is offered: [Retry] [Switch to on-device] [Dismiss].

### Out-of-Memory Kill (OOM)

Android may kill the foreground service process if the model consumes too much RAM. Detected
by observing that `ModelRuntime.isLoaded` returns `false` unexpectedly.

- The conversation session is considered ended (ephemeral memory lost)
- Tier 2 (SQLite) retains all messages written before the kill
- On re-entry, the session reloads from Tier 2 history
- Notification: "Karmik was stopped by the system (low memory). Your conversation was saved."
- Suggestion: offer to switch to a smaller model

### Context Overflow

Detected when token count approaches `contextWindowTokens`. Handled proactively by rolling
context compression (see `specs/08-memory.md`). If compression itself fails (e.g., model
returns empty output):

- Truncate oldest messages (hard truncation) to reach 70% capacity
- Log truncation event to audit log
- Continue inference — user is not interrupted

### Corrupt Model File

Detected when `llama_model_load()` returns an error code (bad magic bytes, truncated file).

- Mark model as `failed` in `karmik_models.db`
- Surface: "Model file appears to be corrupted. Re-download it in Settings → Models."
- Offer alternative downloaded models if any are available

---

## Tool Failures

Every tool returns a `ToolResult`. When `success: false`, the result's `error` field is
injected into the agent's context as a tool result message. The agent can:
- Retry the same tool with different parameters
- Use an alternative tool
- Ask the user for help
- Abandon the subtask and explain why

**Max tool retries**: The agent executor limits identical consecutive tool calls (same tool,
same parameters) to 2 retries before breaking the loop and surfacing an error to the user.
This prevents infinite retry loops.

### Permission denied

If `PermissionChecker.canExecute()` returns false, the tool result is:
```json
{
  "success": false,
  "error": "Permission denied: this agent is not allowed to use [tool_id]. Enable it in Agent Settings → Tools.",
  "resultType": "text"
}
```

The agent should not retry permission-denied failures — they require user action.

### Android permission not granted (OS level)

If the required Android permission (e.g., `READ_CONTACTS`) is not granted at the OS level:
```json
{
  "success": false,
  "error": "Android permission [PERMISSION_NAME] not granted. Grant it in Settings → Apps → Karmik → Permissions.",
  "resultType": "text"
}
```

A deep-link action is surfaced in the overlay: "Grant permission →".

---

## Network Failures (Remote Providers)

### 401 Unauthorized (invalid API key)

- Stop inference immediately
- Surface: "Invalid API key for [Provider]. Update it in Settings → Providers."
- Deep-link to provider settings screen
- Do not retry (retrying with a bad key wastes quota and may trigger provider rate limits)

### 429 Too Many Requests (rate limit)

- Extract `Retry-After` header if present (providers like Anthropic and Groq include it)
- If `Retry-After ≤ 10s`: auto-retry silently after the specified delay
- If `Retry-After > 10s` or header absent: notify user with wait time estimate, offer [Switch to on-device]
- Log to audit log with provider name and retry delay

### 404 Model Not Found

- Provider's model ID is no longer valid (model deprecated, renamed)
- Fall back to the provider's default model (e.g., provider's latest Flash variant for Gemini)
- Notify user: "Model [id] is no longer available. Switched to [fallback_model]. Update your provider settings."

### Timeout / DNS Failure / SSL Error

- After 30 seconds with no response: surface "[Provider] is not reachable. Check your internet connection."
- Offer: [Retry] [Switch to on-device] [Cancel]
- Do not retry automatically for DNS/SSL failures (likely persistent)

---

## Background Service Failures

### Agent executor crash (unhandled exception in isolate)

- WorkManager catches isolate exit; logs error to `audit_log` with stack trace summary
- WorkManager retry policy: exponential backoff (30s, 2m, 8m), max 3 attempts
- After 3 failed attempts: deliver notification "Background agent [name] failed to complete its task. Open Karmik to retry."
- The trigger that fired the session is not deleted — it will fire again on next schedule

### Persistent foreground service crash (process killed repeatedly)

If the foreground service crashes 3 times within 5 minutes (detected via launch counter
persisted in `karmik_models.db`):
- Disable automatic service restart temporarily (10-minute cooldown)
- Notify user: "Karmik's background service stopped unexpectedly. Tap to restart."
- Reset crash counter after a successful 10-minute run

---

## Plugin Script Errors

Plugin scripts run in sandboxed Dart isolates. Errors are isolated from the main process.

### Script throws unhandled exception

```dart
// The plugin runner catches all exceptions from the isolate
ToolResult(
  success: false,
  error: "Plugin script error in [plugin_id]: ${e.toString().substring(0, 200)}",
  resultType: "text",
)
```

- Error is logged to audit log (plugin ID, script name, error summary — not full stack trace)
- The agent receives the error result and can reason about it
- The plugin is not disabled automatically on a single failure

### Script hangs (execution timeout)

Plugin scripts have a 10-second execution timeout. If the isolate does not return within
10 seconds, it is killed via `Isolate.kill(priority: Isolate.immediate)`.

Result sent to agent:
```json
{ "success": false, "error": "Plugin script [script_name] timed out after 10 seconds." }
```

### Repeated plugin failures (auto-disable threshold)

If a plugin script fails 5 times in a single session, the plugin is automatically disabled
for that agent session (not globally). The agent is told: "Plugin [name] has been suspended
for this session due to repeated errors."

The user is notified via the overlay peek card: "Plugin [name] had repeated errors and was
suspended. Review it in Settings → Plugins."

---

## Database Errors

### SQLite open failure (corruption, Keystore key lost)

If the primary database (`karmik.db`) cannot be opened:
1. Attempt to open without encryption (detects if key was lost vs. corruption)
2. If key lost (factory reset, device migration): data is unrecoverable by design
   (see `specs/11-privacy-and-security.md`). Show: "Your Karmik data is encrypted to this
   device and cannot be recovered after a factory reset. Starting fresh."
3. If corruption: attempt SQLite integrity check; if fails, rename the corrupt file to
   `karmik.db.bak` and start with an empty database. Offer: "Database was corrupted and
   could not be recovered. A backup file was saved."

**Fallback mode**: If the database cannot be opened and recovery fails, Karmik runs in
**ephemeral mode** — inference works but nothing is persisted. The Home screen shows a
banner: "Running in temporary mode — history and memory are disabled. Fix in Settings → Privacy."

### Write failure (disk full)

If an `INSERT` or `UPDATE` fails due to `SQLITE_FULL`:
- Surface: "Storage is full. Karmik cannot save your conversation. Free up space to continue."
- Inference is allowed to continue in the current session (ephemeral tier still works)
- Writes are retried on the next user interaction after the user dismisses the banner

---

## Model Download Corruption

Detected at the end of the download when SHA-256 verification fails.

1. Delete the partial/corrupt `.gguf` file immediately (do not leave it on disk)
2. Update download state to `failed` in `karmik_models.db`
3. Auto-retry up to 3 times with a fresh download
4. After 3 failures: surface "Download of [model name] failed checksum verification 3 times.
   This may indicate a network problem or a corrupt file on the source server."
5. Offer: [Try again] [Choose a different model]

---

## Error Surface Summary

| Error type | Agent sees it | User sees it | Audit log |
|---|---|---|---|
| Tool failure (`success: false`) | ✓ (in context) | Only if agent asks user for help | ✓ |
| Inference timeout | ✗ | ✓ (banner + actions) | ✓ |
| OOM kill | ✗ | ✓ (notification) | ✓ |
| Context overflow (compression fails) | ✗ (transparent) | ✗ | ✓ |
| Corrupt model | ✗ | ✓ (banner) | ✓ |
| 401 invalid API key | ✗ | ✓ (banner + deep-link) | ✓ |
| 429 rate limit (auto-retry) | ✗ | ✗ (silent) | ✓ |
| 429 rate limit (long wait) | ✗ | ✓ (notification) | ✓ |
| Plugin script crash | ✓ (as tool error) | Only if repeated failures | ✓ |
| Plugin timeout | ✓ (as tool error) | ✗ | ✓ |
| DB open failure | ✗ | ✓ (banner, ephemeral mode) | — (DB unavailable) |
| Disk full | ✗ | ✓ (banner) | ✓ |
| Download checksum fail | ✗ | ✓ (banner + actions) | ✓ |
| Background service crash | ✗ | ✓ (notification after 3 retries) | ✓ |
