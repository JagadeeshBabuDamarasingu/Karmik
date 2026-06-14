# Karmik — Plugin System

## Overview

Plugins are Cursor-style packages: a bundle of **skills**, **rules**, **prompts**, and **scripts**
that extend what agents can do. Installing a plugin gives agents new capabilities without
modifying the core app.

A plugin is a single installable unit. It declares what it provides and what permissions it needs.
The user approves on install; individual agents opt-in to specific plugins.

## Plugin Structure

```
my-plugin/
├── manifest.json          # Required. Metadata, permissions, entry points.
├── skills/
│   ├── search-web.json    # Tool definition (schema + implementation pointer)
│   └── send-email.json
├── rules/
│   ├── always-cite.md     # A rule applied to agent system prompt
│   └── no-pii.md
├── prompts/
│   ├── summarize.md       # Reusable prompt template with {{variable}} slots
│   └── extract-tasks.md
└── scripts/
    ├── parse-calendar.dart # Dart script (runs in sandboxed isolate)
    └── fetch-weather.dart
```

## manifest.json

```json
{
  "id": "com.example.my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "Does useful things",
  "author": "Jane Dev",
  "minKarmikVersion": "1.0.0",
  "permissions": [
    "tools.http.get",
    "tools.calendar.read",
    "android.permission.INTERNET"
  ],
  "provides": {
    "skills": ["skills/search-web.json", "skills/send-email.json"],
    "rules": ["rules/always-cite.md", "rules/no-pii.md"],
    "prompts": ["prompts/summarize.md", "prompts/extract-tasks.md"],
    "scripts": ["scripts/parse-calendar.dart"]
  },
  "entryPoints": {
    "onInstall": "scripts/setup.dart",
    "onUninstall": "scripts/cleanup.dart"
  }
}
```

## Skills

A skill is a tool definition — it tells the agent what the tool does and how to call it.
The implementation can be:

1. **HTTP endpoint** — the skill makes an HTTP call
2. **Dart script** — runs a bundled `.dart` script in a sandboxed isolate
3. **Built-in tool alias** — maps to an existing tool in the Tool Registry
4. **MCP tool** — routes to an MCP server the plugin declares

```json
{
  "id": "search-web",
  "name": "Search the web",
  "description": "Search the internet for information on a topic",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "The search query" },
      "maxResults": { "type": "integer", "default": 5 }
    },
    "required": ["query"]
  },
  "implementation": {
    "type": "http",
    "method": "GET",
    "urlTemplate": "https://api.search.example.com/search?q={{query}}&n={{maxResults}}",
    "headers": { "Authorization": "Bearer {{env.SEARCH_API_KEY}}" },
    "responseMapping": "$.results[*].snippet"
  }
}
```

```json
{
  "id": "parse-calendar",
  "name": "Parse calendar events from text",
  "description": "Extract structured calendar events from natural language",
  "inputSchema": {
    "type": "object",
    "properties": {
      "text": { "type": "string" }
    },
    "required": ["text"]
  },
  "implementation": {
    "type": "script",
    "script": "scripts/parse-calendar.dart",
    "entryFunction": "parseCalendar"
  }
}
```

## Rules

Rules are markdown documents injected into the agent's system prompt when the plugin is active.
They can be:
- **Always-applied** — injected into every agent that has the plugin enabled
- **Agent-specific** — only injected for agents that explicitly reference the rule by ID

```markdown
<!-- rules/always-cite.md -->
<!-- karmik:rule id="always-cite" apply="always" -->

When providing information retrieved from the web or external sources, always include
the source URL at the end of your response. Format: "Source: [url]"
```

```markdown
<!-- rules/no-pii.md -->
<!-- karmik:rule id="no-pii" apply="agent:research-agent" -->

Never include personally identifiable information (names, phone numbers, email addresses,
physical addresses) in responses unless the user explicitly requests it.
```

## Prompts

Reusable prompt templates with `{{variable}}` placeholders. Agents can invoke these
via the `prompts.use` tool, or they can be wired to UI quick-action buttons.

```markdown
<!-- prompts/summarize.md -->
<!-- karmik:prompt id="summarize" label="Summarize" -->

Summarize the following {{content_type}} in {{max_sentences}} sentences or fewer.
Focus on: {{focus_areas}}.

Content:
{{content}}
```

Usage in agent: the agent calls `prompts.use("summarize", { content_type: "article", ... })`
and gets back the filled template as a string to inject into its next inference call.

## Scripts

Dart scripts run in a sandboxed Dart isolate. They have:
- **No** access to Android system APIs (no FFI to native, no platform channels)
- **No** network access unless `tools.http.*` is in the plugin's declared permissions
- **No** filesystem access outside the plugin's sandboxed data directory
- Access to: `dart:core`, `dart:math`, `dart:convert`, and the Karmik Script API

```dart
// scripts/parse-calendar.dart
import 'package:karmik_scripts/karmik_scripts.dart';

Future<Map<String, dynamic>> parseCalendar(Map<String, dynamic> input) async {
  final text = input['text'] as String;
  // Pure Dart logic — no system access
  final events = extractDates(text);
  return {'events': events};
}
```

## Plugin Installation

**Install sources** (Stage 1):
- Local `.zip` file (sideload from Files app)
- URL install (user pastes a direct download link)

**Install sources** (Stage 2, future):
- Karmik plugin marketplace (curated)
- GitHub URL (auto-download from releases)

**Install flow**:
1. App unpacks and validates the plugin zip
2. `manifest.json` is parsed and validated
3. Permission review screen shown to user (what the plugin can access)
4. User approves → plugin installed to app-private storage
5. `onInstall` script runs (if declared) in sandbox
6. Plugin appears in Plugin Manager; user enables it per-agent

**Uninstall flow**:
1. `onUninstall` script runs (cleanup)
2. Plugin data directory deleted
3. Plugin removed from all agent configs
4. References in agent history remain (for audit purposes) but tool calls will fail gracefully

## Plugin Isolation Guarantees

| Resource | Plugin Script Access |
|---|---|
| Network | Only if `tools.http.*` declared in manifest |
| Calendar | Only if `tools.calendar.*` declared |
| SMS / Contacts | Only if `tools.sms.*` / `tools.contacts.*` declared |
| Files (plugin dir) | Always (own sandbox dir only) |
| Files (user files) | Only if `tools.files.*` declared |
| Camera / Mic | Only if declared AND user grants at runtime |
| Other plugins | No cross-plugin access |
| Karmik internals | No (agent configs, memory, audit log are off-limits) |

## Version Compatibility

Plugin manifests declare `minKarmikVersion`. If the installed Karmik version is older, the plugin
is rejected at install time with a clear message. Breaking changes to the plugin API are tracked
with a semantic version bump in Karmik's plugin API version.
