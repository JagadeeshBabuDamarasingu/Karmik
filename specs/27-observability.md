# Karmik — Observability & Debugging

## Overview

Karmik is an agentic system: agents make decisions, call tools, and take actions. When
something goes wrong — or when users want to understand *why* an agent did something — they
need visibility into the agent's reasoning process.

This spec covers the execution trace system, memory retrieval debugging, a performance
dashboard, and a token budget visualizer.

All observability data is **local-only**. No traces are sent to any remote service.

---

## Execution Traces

### What Is an Execution Trace?

An execution trace is a structured record of everything that happened during one agent turn:

```
Turn
  ├── Thought: "The user wants to know what emails arrived while they were offline."
  ├── Tool call: email.list { since: "2026-06-14T09:00:00Z" }  ← 342ms
  │     Result: [3 emails: Alice (meeting), Bob (invoice), Spam]
  ├── Thought: "The invoice from Bob looks important. I should highlight it."
  ├── Memory retrieval: query="Bob invoice"  ← 12ms
  │     Retrieved: [0.91] "Bob's payment terms are Net-30", [0.78] "Bob sent invoice in May"
  └── Final response: "You received 3 emails. The most important is..."
        Tokens: 312 input + 89 output = 401 total | Cost: $0.000 (local) | Time: 1.2s
```

### Trace Data Model

```dart
class ExecutionTrace {
  final String id;
  final String sessionId;
  final String agentId;
  final String messageId;         // the user message that triggered this turn
  final DateTime startedAt;
  final Duration totalDuration;
  final int inputTokens;
  final int outputTokens;
  final double? cost;             // null for local models
  final List<TraceStep> steps;
}

sealed class TraceStep {}

class ThoughtStep extends TraceStep {
  final String thought;
  final Duration duration;
}

class ToolCallStep extends TraceStep {
  final String toolId;
  final Map<String, dynamic> input;
  final dynamic result;
  final bool success;
  final Duration duration;
}

class MemoryRetrievalStep extends TraceStep {
  final String query;
  final List<MemoryCandidate> retrieved;
  final List<MemoryCandidate> rejected;  // top candidates that were NOT used
  final Duration duration;
}

class FinalResponseStep extends TraceStep {
  final String response;
  final int inputTokens;
  final int outputTokens;
  final Duration ttft;            // time to first token
  final Duration totalDuration;
}

class MemoryCandidate {
  final String content;
  final double score;
  final bool selected;
}
```

### Trace Storage

Traces are stored in `karmik_traces.db` (new, separate from `karmik.db`):

```
traces         — ExecutionTrace records (lightweight header)
trace_steps    — TraceStep records (linked to trace by id)
```

**Retention**: 30 days (configurable: 7 / 14 / 30 / 90 / forever). Auto-deleted by a
nightly job. User can also delete all traces via Settings → Observability → "Clear traces".

**Storage size estimate**: ~5KB per trace × 100 turns/day × 30 days = ~15MB. Manageable.

**Encryption**: `karmik_traces.db` is SQLCipher-encrypted (same as other databases).

### Traces in the Chat UI

Each assistant message in the chat has a "trace" icon (⏱️) in the overflow menu. Tapping it
opens the trace detail panel.

**Collapsed view** (default):

```
⏱️ 1.2s · 401 tokens · 3 tool calls  [expand ▾]
```

**Expanded trace view**:

```
Execution Trace
─────────────────────────────────────────────────
  ⚙ email.list { since: "..." }           342ms ✓
     → 3 emails returned
  🧠 Memory: "Bob invoice"                  12ms
     → [0.91] "Bob's payment terms are Net-30"
     → [0.78] "Bob sent invoice in May"
  ✍ Response generated                    820ms
     → 89 tokens · TTFT: 180ms
─────────────────────────────────────────────────
  Total: 1.2s  |  Input: 312  |  Output: 89
  Cost: $0.000 (Phi-3.8B, local)
─────────────────────────────────────────────────
  [View full trace]  [Export JSON]
```

---

## Memory Retrieval Debugger

When viewing a conversation turn's trace, users can see exactly which memories were
retrieved, why, and which were considered but rejected.

```
Memory Retrieval — query: "Bob invoice"
──────────────────────────────────────────────
  USED (top-3):
  [0.91] "Bob's payment terms are Net-30"   · stored 3 days ago
  [0.78] "Bob sent invoice #1042 in May"    · stored 6 weeks ago
  [0.71] "Bob's company is Acme Corp"       · stored 2 months ago

  NOT USED (shown for context):
  [0.62] "Bob and I met at the conference"  · stored 4 months ago
  [0.58] "Alice mentioned Bob's project"    · stored 1 week ago
──────────────────────────────────────────────
  Search took: 12ms | Vectors searched: 847
```

This helps users understand and improve their memory storage (e.g., "that memory has a low
score because it doesn't mention the invoice explicitly").

---

## Tool Call Timeline

The trace panel also includes a horizontal timeline view showing parallel vs. sequential
tool calls and their relative durations:

```
Timeline (total: 1.2s)
│
├─ email.list         ████████████████ 342ms
│
├─ Memory search      ██ 12ms
│
└─ LLM generation     ████████████████████████████████ 820ms
```

For Plan-and-Execute mode with parallel subtasks, the timeline shows concurrent execution:

```
│ s1: browser.fetch [competitor A]  ████████████████████ 800ms
│ s2: browser.fetch [competitor B]  ██████████████████ 720ms
│ s3: browser.fetch [competitor C]  ██████████████████████ 880ms
│ s4: synthesize (waits for s1-s3)  ──────────────────────████████ 600ms
```

---

## Agent Performance Dashboard

Settings → Observability → Performance Dashboard (or a dedicated tab in the agent's settings):

```
Git Assistant · Performance
══════════════════════════════════════════════

Last 7 days
  Total turns:      48
  Avg response:     2.1s
  Avg tokens/turn:  524
  Tool success rate: 96%  (2 failures)
  Estimated cost:   $0.00 (100% local)

Most used tools (last 30 days)
  1. git.diff          ████████████ 34 calls
  2. git.status        ████████████ 31 calls
  3. github.list_prs   ████████ 22 calls
  4. git.log           ████████ 19 calls
  5. memory.search     █████ 15 calls

Recent errors
  git.diff: "Not a git repository" (2 occurrences)
  github.list_prs: "Rate limit exceeded" (1 occurrence)

Response time trend (7 days)
  Mon  Tue  Wed  Thu  Fri  Sat  Sun
  2.3  1.9  2.1  2.4  2.0  1.8  2.1  (seconds)

══════════════════════════════════════════════
```

**Data source**: computed from the `traces` table — no extra tracking required.

---

## Token Budget Visualizer

The context window indicator (already specified in spec 10-ui.md as a small bar in the chat
header) is expanded with a breakdown popover. Tapping the indicator shows:

```
Context Window  (42% used of 128K)
──────────────────────────────────────────────
  System prompt          2,100 tokens  (1.6%)
  Pinned context           580 tokens  (0.5%)
  Plugin rules             120 tokens  (0.1%)
  Memory injection         840 tokens  (0.7%)
  Conversation history  41,200 tokens (32.2%)
  Current turn           9,100 tokens  (7.1%)
  ────────────────────────────────────
  Total used            53,940 tokens (42.1%)
  Available             74,060 tokens (57.9%)
──────────────────────────────────────────────
  At current rate, context will fill in ~14 more turns.
  Rolling compression will activate at 102K tokens.
```

This helps users understand why old messages are being summarized and lets them make
informed decisions about memory configuration.

---

## Observability Settings (Settings → Observability)

```
Observability
══════════════════════════════════════════════

Execution traces
  Enable traces          [✓ On]
  Retain for            [30 days ▾]
  Show trace in chat    [✓ On]
  Storage used:         12.4 MB   [Clear all traces]

Memory retrieval debug
  Show retrieved memories in trace  [✓ On]
  Show rejected candidates          [Off]   ← can be noisy

Performance dashboard
  [View dashboard →]

Token budget
  Show context bar in chat  [✓ On]
  Tap for breakdown         [✓ On]

══════════════════════════════════════════════
```

---

## Log Export

Traces can be exported in JSON format for external analysis (power users, developers):

```
Settings → Observability → Export traces
  Date range: [Last 30 days ▾]
  Agent: [All agents ▾]
  [Export JSON]
```

The exported JSON follows the `ExecutionTrace` schema. It can be loaded into local analysis
tools (Python/Jupyter, SQLite) or shared with Karmik developers to debug issues.

---

## Platform Availability

All observability features are available on all platforms where Karmik runs local inference
(Android, iOS, macOS, Windows, Linux). On Web (remote-only), traces are available but cost
data comes from the remote provider's token counts, and memory retrieval data is limited
(semantic search runs locally even on Web).
