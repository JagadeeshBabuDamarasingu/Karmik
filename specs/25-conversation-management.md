# Karmik — Conversation Management

## Overview

Conversations are currently per-agent silos. This spec adds global search across all agent
conversations, organization tools (tagging, starring, pinning), multi-format export, import
from other AI apps, auto-archiving, and read-only shareable links.

---

## Data Model Changes

New fields on the `Session` record in `karmik.db`:

```dart
class Session {
  // ... existing fields (id, agentId, createdAt, updatedAt, messageCount) ...
  final String? title;           // auto-generated or user-set title
  final bool starred;            // user-starred for quick access
  final bool pinned;             // pinned to top of list
  final bool archived;           // soft-archived (hidden from main list)
  final List<String> tags;       // user-set tags (stored as JSON array)
  final String? summary;         // AI-generated 1-sentence summary (created lazily)
}
```

**Auto-title generation**: after 3+ messages in a new session, Karmik runs a lightweight
inference pass (using the configured fast model) to generate a 5–8 word title for the
conversation. Stored in `sessions.title`. User can rename it.

**Auto-summary generation**: generated alongside the title. Stored in `sessions.summary`.
Used as the search result snippet and in the conversation list.

---

## Global Search

A new **Search** tab (or `Cmd+K` / `Ctrl+K` on desktop) searches across all conversations
from all agents.

### Search Modes

| Mode | Implementation |
|---|---|
| Full-text | FTS5 over `messages.content` — fast, exact keyword matches |
| Semantic | sqlite-vec cosine similarity over `sessions.summary_embedding` |
| Hybrid | FTS5 score + semantic score combined (default) |

### Search UI

```
Search conversations
──────────────────────────────────────────────
🔍 [npm install error                       ]

Filters: [Agent ▾]  [Date ▾]  [Tag ▾]  [★ Starred]

Results (12 matches):
────────────────────
📎 Git Assistant · 3 hours ago
   "npm install error in the project"
   ...you ran npm install and got "ENOENT: no such file or directory"...

📎 Shell Assistant · Yesterday
   "Node.js setup troubleshooting"
   ...the error was caused by a missing node_modules symlink...

📎 Research · 2 days ago
   "npm vs pnpm performance comparison"
   ...
────────────────────
```

### Search Scope Controls

- Filter by agent, date range, tag, or starred status
- Search within a specific agent's conversations only (from that agent's chat view)
- Search within the current conversation (Cmd+F while in a chat)

---

## Conversation Organization

### Tags

Users can add free-text tags to any conversation. Tags are displayed as colored chips in the
conversation list. Suggestions are auto-generated based on conversation topic.

**Tag management**: Settings → Conversations → Manage Tags (rename, merge, delete).

### Starring

The star icon (`★`) in the conversation list marks a conversation for quick access.
Starred conversations appear in a "Starred" section at the top of the chat list.

### Pinning

Pinning keeps a conversation permanently at the top of its agent's chat list (above
starred items). Maximum 3 pinned conversations per agent.

### Archiving

Users can archive conversations to hide them from the main list without deleting them.
Archived conversations appear in a separate "Archive" section accessible from the
conversation list overflow menu.

**Auto-archive**: in Settings → Conversations → Auto-archive, users can configure:
- Archive conversations older than: 30 / 60 / 90 / 180 / never days
- Archive conversations with no activity for: 7 / 14 / 30 / never days

---

## Export

### Multi-Format Export

A conversation can be exported from the `···` overflow menu → "Export":

```
Export Conversation
─────────────────
Format:
  (●) Markdown (.md)
  ( ) HTML (.html)       — includes CSS for clean rendering
  ( ) JSON (.json)       — full metadata: timestamps, tool calls, model, costs
  ( ) PDF (.pdf)         — formatted, printable

Include:
  [✓] Tool call details
  [✓] Thinking steps (if extended thinking was used)
  [✗] Raw model tokens

              [Export]
```

**Markdown export** (existing, enhanced):
```markdown
# "npm install error in the project"
Agent: Git Assistant | Date: 2026-06-14 | Model: Phi-3.8B

---

**You** (14:32): I'm getting an npm install error...

**Git Assistant** (14:32):
> Tool: shell.run `npm install`
> Exit code 1: ENOENT...

Let me check what's happening...
```

**HTML export**: self-contained single-file HTML with embedded CSS (dark/light modes).
No JavaScript — pure static HTML for archiving.

**JSON export**: full fidelity — every message, every tool call input/output, timing,
token counts, model used, cost.

**PDF export**: rendered via Flutter's `printing` package (uses the platform PDF renderer).
Includes a cover page with metadata.

### Bulk Export

Settings → Conversations → Export All:
- Select which agents' conversations to include
- Choose format (Markdown or JSON)
- Downloads as a `.zip` archive

---

## Import

Users can import conversations from other AI apps:

### Supported Import Formats

| Source | Format | Implementation |
|---|---|---|
| ChatGPT | `conversations.json` (exported from ChatGPT settings) | Parse OpenAI export schema |
| Claude (Anthropic web) | `conversations.json` (exported from claude.ai) | Parse Anthropic export schema |
| Karmik JSON export | `karmik_export_*.json` | Lossless round-trip |
| Generic JSON | `[{ "role": "user"/"assistant", "content": "..." }]` | Basic import, no metadata |

**Import flow**:
1. Settings → Conversations → Import → "Choose file"
2. Karmik detects the format
3. Preview: shows first 3 messages + total count
4. User selects target agent (default: create a new agent named "Imported from ChatGPT")
5. Import runs; conversations appear in the agent's history

**Limitations**: imported conversations have no tool call data (only chat messages).
They are stored as regular messages, not re-runnable tool call sessions.

---

## Shareable Links (Read-Only)

Users can generate a read-only static HTML snapshot of a conversation to share with others.

**Privacy model**: this is a local export only — no server involved. The generated HTML is
saved to a file and the user shares it manually (attach to email, host on their own server,
etc.). There is no "Karmik link" that resolves to a live server.

**How it works**:
- Same as HTML export (above), but with a note at the top: "Shared by [user's display name] via Karmik"
- File is saved to the Downloads folder
- Share sheet opens with the file attached (on mobile) or "Copy file path" on desktop
- Optional: if user has a personal website or GitHub Pages, they can host the file there

**Why no server-based links**: Karmik is local-first. Adding a link-sharing server would
require cloud infrastructure and raise privacy questions. The local HTML file approach gives
users full control.

---

## Conversation List UI Updates

The existing chat list per agent gets:
- Star icon (★) in each row (tap to toggle)
- Tag chips displayed under the conversation title
- Swipe-left: Archive, Delete
- Swipe-right: Star
- Long-press context menu: Star, Pin, Tag, Export, Archive, Delete

Global conversation list (new "All conversations" view in the Home tab):
- Groups by: Today / Yesterday / This week / This month / Earlier
- Sorted by: Last activity (default) | Created | Starred first
- Filter bar: agent, tag, starred, archived

---

## Storage Impact

**Summaries and embeddings**: auto-generated summaries are ~50 tokens per conversation.
Summary embeddings (384-dim float32) are ~1.5KB per conversation. For 1000 conversations,
that's ~1.5MB of additional vector data — negligible.

**FTS5 index**: a new FTS5 virtual table `messages_fts` indexes `messages.content`. For
10,000 messages averaging 100 words each, the FTS index is ~5–10MB. Updated incrementally
as new messages are added.

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| Global search | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Tagging / starring / pinning | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Auto-archive | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Export (Markdown, JSON, HTML) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Export (PDF) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Bulk export | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Import (ChatGPT/Claude JSON) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Shareable HTML link | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Auto-title generation | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
