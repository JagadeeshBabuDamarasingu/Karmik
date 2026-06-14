# Karmik — Automation Workflows & Webhooks

## Overview

The existing trigger system (spec 09) fires one agent when one condition is met. This spec adds:

1. **Condition expressions**: triggers fire only when a DSL condition evaluates to true
2. **Trigger chaining**: one trigger fires an agent that can fire another trigger
3. **Multi-step workflows**: visual automation builder (like Shortcuts/IFTTT but local-first)
4. **Inbound webhooks**: external services POST to Karmik → agent fires
5. **Outbound webhooks**: agents POST to external URLs as an action
6. **iOS Shortcuts integration**: Karmik actions available in Apple Shortcuts + Siri
7. **Android Tasker / Automate plugin**: Karmik exposed as a plugin for Android automation apps

---

## Condition DSL

Every trigger now has an optional `condition` field — a simple expression evaluated against
the trigger's event data before deciding whether to fire the agent.

### Syntax

```
{{ <expression> }}
```

Expressions support:
- Variables from the trigger payload (e.g., `steps`, `battery`, `hour`, `app`, `text`)
- Comparison operators: `>`, `<`, `>=`, `<=`, `==`, `!=`
- Boolean operators: `AND`, `OR`, `NOT`
- Parentheses for grouping
- String literals: `"whatsapp"`, `"urgent"`
- Number literals: `8000`, `20`
- Time functions: `hour()`, `minute()`, `dayOfWeek()`, `date()` — all in local time
- `contains(variable, "string")` — substring check
- `matches(variable, "regex")` — regex match

### Variables by Trigger Type

| Trigger type | Available variables |
|---|---|
| ScheduleTrigger | `hour`, `minute`, `dayOfWeek` (1=Mon…7=Sun), `date` (YYYY-MM-DD) |
| NotificationTrigger | `app`, `title`, `body`, `package` |
| HealthTrigger | `steps`, `sleepMinutes`, `heartRate`, `battery` |
| LocationTrigger | `event` ("enter" or "exit"), `latitude`, `longitude` |
| SmartHomeTrigger | `deviceId`, `attribute`, `value`, `previousValue` |
| FocusTrigger | `minutesCompleted`, `completedSessions` |
| WakeWordTrigger | `transcript`, `confidence` |
| WebhookTrigger | `payload.*` (any field from the POST body) |

### Examples

```
# Only run Morning Briefing on weekdays
condition: {{ dayOfWeek() >= 1 AND dayOfWeek() <= 5 }}

# Alert only if motion sensor fires at night
condition: {{ value == "active" AND (hour() >= 22 OR hour() < 6) }}

# Only capture WhatsApp messages containing task keywords
condition: {{ app == "com.whatsapp" AND (contains(body, "todo") OR contains(body, "remind")) }}

# Wellness check only if steps < goal and it's after 8pm
condition: {{ steps < 8000 AND hour() >= 20 }}

# Fire webhook if temperature drops below threshold
condition: {{ attribute == "temperature" AND value < 18 }}
```

### Condition Evaluation

Conditions are evaluated in the `TriggerEngine` (foreground service) before spawning an
agent. Failed conditions are logged as `"trigger_skipped": true` in the audit log (not as
errors — skipping is expected behavior).

---

## Trigger Chaining

A trigger can optionally specify a follow-up trigger to fire after the agent completes.
This enables linear automation pipelines.

```dart
class TriggerChain {
  final String nextTriggerId;      // ID of the next trigger to fire
  final String? condition;         // optional condition on the agent's output
  final Map<String, String> passOutputAs;  // map agent output → next trigger payload vars
}
```

**Example**: Schedule → Morning Briefing → (if briefing mentions a meeting) → Meeting
Notes agent (starts recording).

For branching ("if agent output contains X, fire agent A; else fire agent B"), the agent
itself calls `agents.spawn` (already in the tool registry) to conditionally spawn sub-agents.

---

## Automation Workflow Builder (UI)

A visual workflow editor in Settings → Automations:

```
Automations
══════════════════════════════════════════════

+ Evening Wind-Down                          [Edit] [▶ Run] [✕]
  ┌─────────────────────────────────────┐
  │ TRIGGER                             │
  │ Schedule · 9:00 PM                  │
  │ Condition: {{ dayOfWeek() <= 5 }}   │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │ AGENT: Focus Coach                  │
  │ Task: "Review today's focus stats   │
  │ and suggest tomorrow's plan"        │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │ ACTION: webhook.send                │
  │ URL: https://webhook.site/xxx       │
  │ Body: {{ agent.output }}            │
  └─────────────────────────────────────┘

[+ Add automation]
══════════════════════════════════════════════
```

Each automation has:
- A **trigger** (any existing trigger type + condition expression)
- One or more **steps** (agent + task description, or a direct tool action)
- An optional **output action** (webhook, notification, or chained trigger)

---

## Inbound Webhooks

Karmik can receive HTTP POST requests from external services (IFTTT, Zapier, n8n, Home
Assistant, etc.) and fire a configured agent in response.

### Webhook Endpoint

```
POST http://{device-ip}:5176/webhook/{webhookId}
Authorization: Bearer {webhookToken}
Content-Type: application/json

{ "any": "payload", "you": "want" }
```

- **Port**: 5176 (separate from the MCP server on 5175; configurable)
- **Webhook ID**: UUID, auto-generated per webhook configuration
- **Auth token**: per-webhook Bearer token (stored locally, shown in UI on creation)
- **Payload**: any JSON — available in condition DSL as `payload.*` variables

### Webhook Configuration

```dart
class WebhookTriggerConfig {
  final String id;              // UUID (becomes the URL path segment)
  final String displayName;
  final String? authToken;      // required Bearer token; null = unauthenticated (not recommended)
  final String agentId;         // agent to fire
  final String taskTemplate;    // task description; can reference {{ payload.field }}
  final String? condition;      // optional condition expression
}
```

**Task template example**:
```
"A new order was received from {{ payload.customer_name }} for {{ payload.item }}. 
 Add it to my task list and send a confirmation email."
```

### Privacy Mode

Inbound webhooks bind to the LAN IP (or localhost if on the same machine). In Privacy Mode:
- Webhooks from localhost: allowed
- Webhooks from LAN: allowed
- Public internet inbound: the webhook port is not opened to the internet (no port forwarding
  required for IFTTT/Zapier; they must use a Karmik relay or self-hosted tunnel)

**Zapier/IFTTT note**: these services require a public URL. Options:
1. Use a self-hosted tunnel (ngrok, Tailscale funnel, Cloudflare Tunnel) — user's responsibility
2. The Karmik Sync Server (spec 21) can optionally proxy webhooks to the local device

---

## Outbound Webhooks

The `webhook.send` tool lets agents POST to any external URL:

```
webhook.send
  → POST JSON data to an external URL
  Input: {
    "url": string,
    "payload": {},
    "headers"?: {},
    "method"?: "POST|PUT|PATCH"  // default POST
  }
  Output: { "status": int, "body": string }
  Co-pilot: first use per URL (user approves "allow Karmik to POST to https://...")
```

**Privacy Mode**: `webhook.send` is blocked (internet call). Allowed if URL is a LAN IP.

**Permission**: `tools.webhook.send` off by default. Must be explicitly granted.

---

## iOS Shortcuts Integration

Karmik actions are available as actions in the **Apple Shortcuts** app on iOS and macOS,
invocable via Siri ("Hey Siri, ask Karmik to…").

**Implementation**: Karmik registers as a Shortcuts provider via `AppIntents` framework
(iOS 16+, macOS 13+).

### Exposed Shortcut Actions

| Action | Parameters | Returns |
|---|---|---|
| Ask Karmik | agent (optional), message | response text |
| Add to Reading List | url | saved item title |
| Create Task | title, due date (optional) | task id |
| Store Memory | content, tags | memory id |
| Search Memory | query | top matching memories |
| Run Agent | agent name, task | result |

### Siri Integration

Users can create Shortcuts that call Karmik and invoke them with Siri:

```
"Hey Siri, add this to my Karmik reading list"
  → Shortcuts: Add to Reading List (URL from clipboard or shared sheet)
  → Karmik: readinglist.add { url: "..." }
  → Siri speaks: "Saved to your reading list"
```

**Siri Shortcuts donation**: Karmik donates `INIntent` objects for common actions after
the user performs them, so Siri proactively suggests them.

---

## Android Tasker Plugin

Karmik exposes itself as a **Tasker** plugin (and compatible automation apps: Automate,
MacroDroid, Locale) via the Tasker Plugin API (`com.joaomgcd.taskerpluginlibrary`).

### Plugin Actions (Karmik → Tasker)

Tasker can invoke:
- **Ask Agent**: provide agent name + message → returns response
- **Create Task**: title + optional due date
- **Store Memory**: content + tags
- **Run Automation**: trigger a named Karmik automation by name

### Plugin Events (Tasker ← Karmik)

Karmik can send events to Tasker when:
- An agent completes a task (broadcasts `com.karmik.agent.COMPLETED` intent)
- A Karmik trigger fires (broadcasts `com.karmik.trigger.FIRED` with trigger data)

This enables Tasker to react to Karmik events (e.g., "when Karmik completes the Morning
Briefing, enable Wi-Fi hotspot").

**Setup**: user installs "Karmik Plugin for Tasker" from Play Store (or as an APK bundled
with Karmik). In Tasker: Action → Plugin → Karmik → pick action.

---

## Webhook & Automation Permissions

| Permission | Default | Notes |
|---|---|---|
| `tools.webhook.send` | Off | Must be explicitly granted per agent |
| Inbound webhooks | Off | Enabled per webhook config; requires token auth |
| iOS Shortcuts | Auto (on iOS/macOS) | Always available; no extra grant needed |
| Tasker plugin | Off | Requires Tasker installed + user links apps |

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| Condition expressions on all triggers | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Inbound webhooks | ✓ | ~ (foreground only) | ✓ | ✓ | ✓ | — |
| Outbound webhook.send | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| iOS Shortcuts | — | ✓ | ✓ | — | — | — |
| Android Tasker plugin | ✓ | — | — | — | — | — |
| Automation workflow builder UI | ✓ | ✓ | ✓ | ✓ | ✓ | — |

**iOS inbound webhooks (~)**: the webhook server requires the Karmik app to be in the
foreground (iOS kills background network servers). For reliable webhook delivery on iOS,
use the Karmik Sync Server proxy or use Shortcuts-based triggers instead.
