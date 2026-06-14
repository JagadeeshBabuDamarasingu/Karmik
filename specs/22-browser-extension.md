# Karmik — Browser Extension

## Overview

The Karmik browser extension brings Karmik into any webpage: web clipping, in-page agent
chat, selected-text actions, and form fill from Karmik's memory — without switching apps.

The extension is a thin client. It connects to the Karmik desktop app's MCP server
(`localhost:5173`). Karmik must be running on the same machine for the extension to function.
When Karmik is not running, the extension shows a "Karmik not running — open the app" notice.

---

## Supported Browsers

| Browser | Manifest version | Distribution |
|---|---|---|
| Chrome / Chromium | Manifest V3 | Chrome Web Store |
| Microsoft Edge | Manifest V3 | Edge Add-ons store |
| Firefox | MV2 compatibility shim | Firefox Add-ons (AMO) |
| Safari (macOS) | Safari App Extension (WKWebExtension) | Mac App Store; bundled with Karmik desktop |
| Arc / Brave / Vivaldi | Manifest V3 | Chrome Web Store (same extension) |

**Firefox**: MV3 is supported in Firefox 109+ but has feature gaps. Karmik uses a thin MV2
compatibility layer for Firefox that wraps the MV3 service worker as a background script.

**Safari**: distributed as part of the Karmik macOS app. Users enable it in Safari → Settings
→ Extensions → Karmik. Requires Safari 16.4+ (WKWebExtension support).

---

## Architecture

```
Browser Tab (Content Script)
        │  postMessage
        ▼
Extension Background Service Worker
        │  HTTP (localhost only)
        ▼
Karmik MCP Server (localhost:5173)
        │  native tool calls
        ▼
Karmik Desktop App (agents, memory, tools)
```

**Content script**: injected into every page (with `run_at: document_idle`). Adds the
floating Karmik button, intercepts right-click menu actions, and listens for text selection.
Communicates with the background service worker via `chrome.runtime.sendMessage`.

**Background service worker**: the extension's core. Manages the connection to Karmik's
MCP server. Handles session state (which agent is active, conversation history for the
current tab).

**No backend**: the extension communicates only with `localhost`. No data is sent to any
remote server by the extension itself. All processing happens in Karmik.

**Connection check**: background worker pings `http://localhost:5173/health` on install and
every 30 seconds. If unreachable: extension icon shows a grey badge; features degrade
gracefully with a "Start Karmik" prompt.

---

## Features

### 1. Floating Karmik Button

A small Karmik icon appears in the bottom-right corner of every page (configurable: hide,
change position, or auto-hide when no text is selected).

Clicking it opens the **Karmik sidebar** — a 380px-wide panel that slides in from the right:

```
┌─────────────────────────────────────────────┐
│ ▸ Karmik                        [—] [✕]     │
│─────────────────────────────────────────────│
│ Agent: Work Assistant  ▾                    │
│─────────────────────────────────────────────│
│                                             │
│  You: What is this page about?              │
│                                             │
│  Karmik: This is the Stripe API docs for    │
│  payment intents. The key endpoint is       │
│  POST /v1/payment_intents...                │
│                                             │
│─────────────────────────────────────────────│
│  [Type or @mention...]          [🎤] [→]    │
└─────────────────────────────────────────────┘
```

The sidebar persists as the user navigates (conversation continues across page loads within
the same tab session). Each tab has its own conversation context.

**Page context injection**: when the user sends a message, the current page's title + URL
are automatically included in the agent context. The page text content is included if the
user has granted "read page content" permission for that domain.

### 2. Web Clipper

Clicking "Clip page" (or right-click → "Save to Karmik reading list") triggers:

1. Extension extracts the page's main content (Readability.js for article extraction)
2. Sends URL + content to Karmik via `readinglist.add`
3. Karmik generates a summary and saves to the reading list
4. Extension shows a toast: "✓ Saved to Reading List — 'Title of article'"

**Highlight clip**: select text on a page → right-click → "Clip selection" → saves the
selected text as a note (via `notes.create`) with the source URL as a footnote.

### 3. Right-Click Context Menu

When text is selected on any page, the right-click menu includes:

```
Karmik ▸
  Ask Karmik about this
  Translate to English
  Summarize
  Find similar in notes
  Save as snippet
  Copy improved version
```

Each option:
- **Ask Karmik about this**: opens sidebar with the selected text pre-filled as context
- **Translate to English**: calls `text.translate`, shows result in a tooltip overlay
- **Summarize**: calls agent with "Summarize this: {text}", shows in sidebar
- **Find similar in notes**: calls `notes.search { query: selectedText }`, shows results in sidebar
- **Save as snippet**: calls `snippets.store` with the selected code, prompts for description
- **Copy improved version**: agent rewrites the selected text, copies to clipboard

### 4. Form Auto-Fill from Karmik Context

On pages with `<input>` and `<textarea>` fields, the extension can fill them from Karmik's
memory and context.

**How**: user focuses an input → a small Karmik icon appears at the right edge → clicking it
opens a mini-popup:

```
┌──────────────────────────────────┐
│  Fill from Karmik                │
│  ─────────────────────────────   │
│  Name        → Jagadeesh D.      │
│  Email       → jagadeesh@…       │
│  Company     → Karmik Inc.       │
│  Job title   → Founder           │
│  ─────────────────────────────   │
│  Or type a query: [___________]  │
└──────────────────────────────────┘
```

Field suggestions come from `memory.search { query: fieldLabel }`. Selecting a suggestion
fills the field. The fill happens in the browser — no data goes through Karmik for the actual
input; only the lookup query is sent to Karmik.

**Permission**: requires the user to grant "Read page structure" for the domain (to detect
field labels). Off by default; requested on first use of auto-fill.

### 5. Karmik Popup (Extension Icon Click)

Clicking the extension icon in the browser toolbar opens a compact 340×500px popup:

```
┌─────────────────────────────────────┐
│ Karmik                              │
│ ●  Connected to Karmik on localhost │
│─────────────────────────────────────│
│ Quick actions                       │
│  [Ask about this page]              │
│  [Clip to Reading List]             │
│  [Open sidebar chat]                │
│─────────────────────────────────────│
│ Recent (this tab)                   │
│  2 messages · Work Assistant        │
│                                     │
│ Switch agent: [Work Assistant ▾]    │
└─────────────────────────────────────┘
```

---

## Permissions

The extension requests the following browser permissions:

| Permission | Purpose |
|---|---|
| `activeTab` | Read the current tab's URL and title (on explicit user action only) |
| `contextMenus` | Add right-click menu entries |
| `storage` | Store per-domain permission grants and extension settings |
| `scripting` | Inject content script into pages |
| `clipboardWrite` | Write translated/improved text to clipboard |
| Host: `http://localhost:5173/*` | Connect to Karmik MCP server |

**Optional permissions** (requested on first use of the feature that needs them):

| Permission | Required for |
|---|---|
| Host: `<all_urls>` (read) | "Read page content" — injecting Karmik sidebar on all pages |
| `clipboardRead` | Reading clipboard for "Improve clipboard content" action |

**Per-domain grants**: reading page content beyond the title/URL requires a per-domain grant.
The extension tracks these in `chrome.storage.local`. Users can review and revoke domain grants
in the extension's Options page.

---

## Privacy

- The extension does not collect any telemetry or analytics.
- The extension only connects to `localhost` — no external network requests.
- Page content is only sent to Karmik (local process) when the user explicitly grants a domain.
- Selected text is only sent when the user explicitly invokes a right-click action.
- All data goes to Karmik, which applies its own Privacy Mode rules. If Karmik is in Privacy
  Mode, the extension's agent chat respects it (no remote inference calls).

---

## Extension Settings (Options Page)

```
Karmik Extension Settings
══════════════════════════════════════════════

Connection
  Karmik MCP server:  http://localhost:[5173]
  Status:             ● Connected

Default agent
  [Work Assistant ▾]

Sidebar
  Open sidebar on:    [Manual (button click) ▾]
  Sidebar position:   [Right ▾]
  Float button:       [✓ Show on all pages]

Auto-fill
  Enable auto-fill:   [✓ On]
  [Manage domain grants]

Domains with page-read access
  stripe.com          [Revoke]
  github.com          [Revoke]
  notion.so           [Revoke]
  [Clear all]

══════════════════════════════════════════════
```

---

## Communication Protocol

The extension talks to Karmik via the MCP server using standard MCP tool calls:

```
Extension → POST http://localhost:5173/mcp
  { method: "tools/call", params: { name: "readinglist.add", arguments: { url, content } } }

Extension → POST http://localhost:5173/mcp
  { method: "sampling/createMessage", params: { messages: [...], system: agentSystemPrompt } }
```

For chat (agent conversations), the extension uses MCP's `sampling/createMessage` with the
current agent's system prompt and conversation history maintained in the service worker.

**Auth**: the extension includes the Bearer token from Settings → MCP → Server token. The
token is stored in `chrome.storage.local` (not `sync`, to prevent cross-device token exposure).

---

## Platform Availability

| Feature | macOS | Windows | Linux | Notes |
|---|---|---|---|---|
| Chrome/Edge extension | ✓ | ✓ | ✓ | Same MV3 extension |
| Firefox extension | ✓ | ✓ | ✓ | MV2 compatibility layer |
| Safari extension | ✓ | — | — | Bundled with Karmik macOS app |
| All features | ✓ | ✓ | ✓ | Requires Karmik desktop to be running |

The browser extension is a **desktop-only** feature. It connects to the Karmik desktop app's
local server. Mobile browsers do not support browser extensions in the same way.
