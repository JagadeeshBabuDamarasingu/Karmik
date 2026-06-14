# Karmik — Agent System

## Overview

Agents are the core unit of Karmik. An agent is a named, persistent AI persona with its own:
- Identity (name, avatar, description)
- System prompt (instructions that shape its behavior)
- Memory configuration (what it remembers, for how long)
- Tool permissions (which tools it can use)
- Model preference (which runtime/model to use)
- Orchestration mode (how it executes tasks)

Agents persist across sessions. A user can have multiple agents for different domains
("Work", "Personal", "Research", "Home Automation"), each with isolated or shared memory.

## Agent Definition

```dart
class AgentConfig {
  final String id;                    // UUID, immutable
  final String name;                  // "Aria", "Work Assistant", etc.
  final String? avatarEmoji;          // quick visual identity
  final String description;           // shown in agent list
  final String systemPrompt;          // core behavioral instructions
  final String? modelOverride;        // model ID; null = use global default
  final OrchestrationMode mode;
  final MemoryConfig memoryConfig;
  final List<String> allowedTools;    // tool IDs this agent can use
  final List<String> activePlugins;   // plugin IDs applied to this agent
  final TriggerConfig? triggerConfig; // if agent runs in background
  final DateTime createdAt;
  final DateTime updatedAt;
}

class MemoryConfig {
  final bool useEphemeralMemory;      // always true
  final bool usePersistentHistory;    // conversation log in SQLite
  final bool useLongTermMemory;       // semantic vector store
  final int historyRetentionDays;     // -1 = forever
  final MemoryScope scope;            // isolated | shared
}

enum MemoryScope {
  isolated,   // this agent only sees its own memory
  shared,     // can read from the user's shared memory pool
}
```

## Orchestration Modes

The user selects the orchestration mode per agent (configurable). The mode determines how the
agent executor processes a task.

### Mode 1: ReAct (Reason + Act)

The default mode. The agent loops: reason → act → observe → reason → act → ... until done.

```
System prompt + history + user input
        │
        ▼
[Model] → output: thought + action (tool call) OR final answer
        │
        ├─ Tool call → [Tool Registry] → result → append to context → loop
        │
        └─ Final answer → return to user
```

- Maximum iterations: configurable (default 10, max 25)
- Infinite loop guard: detect identical consecutive tool calls → break
- Best for: open-ended tasks, research, multi-step workflows

### Mode 2: Plan-and-Execute

Two-phase execution. A planner agent first decomposes the task into subtasks, then
executor sub-agents run each subtask (sequentially or in parallel).

```
User task
        │
        ▼
[Planner Agent] (using fast/cheap model) → task decomposition: [subtask1, subtask2, subtask3]
        │
        ▼
[Executor Pool] → spawn one executor per subtask
        │          (run in parallel or sequential based on dependency graph)
        ▼
[Result Aggregator] → merge outputs → final response to user
```

- Planner uses the agent's configured model; executors can use a smaller fast model
- Dependency specification: planner outputs a DAG, not just a flat list
- Best for: complex research tasks, batch operations, long-horizon planning

**DAG format** (planner outputs this JSON, parsed by the `OrchestratorPool`):

```json
{
  "goal": "Research and summarize the top 3 competitors",
  "subtasks": [
    {
      "id": "s1",
      "task": "Search the web for Competitor A's pricing page",
      "dependsOn": []
    },
    {
      "id": "s2",
      "task": "Search the web for Competitor B's pricing page",
      "dependsOn": []
    },
    {
      "id": "s3",
      "task": "Search the web for Competitor C's pricing page",
      "dependsOn": []
    },
    {
      "id": "s4",
      "task": "Synthesize the three pricing pages into a comparison table",
      "dependsOn": ["s1", "s2", "s3"]
    }
  ]
}
```

Rules:
- `dependsOn` lists subtask IDs that must complete before this subtask starts
- Subtasks with empty `dependsOn` are eligible to run in parallel immediately
- If any dependency fails with an unrecoverable error, dependent subtasks are skipped and
  the aggregator notes the failure in the final response
- Max subtask count: 10 (planner is instructed to consolidate if it would exceed this)

### Mode 3: Autopilot

Background, fully autonomous. Agent runs without user interaction until complete.
Result is delivered as a notification.

```
Trigger (schedule / notification / event)
        │
        ▼
[Background Service] → spawn agent
        │
        ▼
[Agent Executor] → ReAct or Plan-Execute loop
        │
        ▼
Result → Android notification → user can review/approve/dismiss
```

- No step-by-step UI; result delivered on completion
- For long-running tasks: intermediate progress notifications at configurable intervals
- Best for: morning briefing, scheduled summaries, proactive task capture

### Mode 4: Co-pilot

Human-in-the-loop. Agent proposes each action before executing it. User taps to approve or modify.

```
User task
        │
        ▼
[Model] → proposed action (tool call with description: "I will send this SMS to Alex...")
        │
        ▼
[UI] → show action card to user → Approve | Edit | Skip | Cancel
        │
        ▼ (if approved)
[Tool Registry] → execute → result → next reasoning step → next proposed action
```

- Best for: destructive or irreversible actions (SMS, email, file deletion)
- Agents that access sensitive tools should default to this mode
- Each proposed action includes: what will be done, why, and what data will be accessed

## Agent Library (Built-in Agents)

Pre-built agents available on first launch. Users can clone and customize.

| Agent | Default Mode | Default Tools | Purpose |
|---|---|---|---|
| Task Capture | ReAct | notifications.list, tasks.create | GTD capture from notifications/voice |
| Morning Briefing | Autopilot | calendar.list_events, tasks.list, smarthome.query_sensor | Daily summary on schedule |
| Meeting Notes | ReAct | microphone.record, tasks.create, calendar.update | Call recording → structured notes |
| Research | Plan-Execute | http.get, browser.fetch, memory.store | Deep research with source tracking |
| Home Automation | ReAct + Co-pilot | smarthome.*, location.current | Smart device control; dangerous commands (lock/camera) always use Co-pilot |
| Focus Coach | ReAct | focus.*, tasks.list, calendar.list_events | Manages Pomodoro sessions, blocks distractions, tracks daily focus stats |
| Daily Journal | ReAct | microphone.record, stt.transcribe, notes.create, memory.store | Voice journaling → structured note with automatic tagging |
| Daily Standup | Autopilot | tasks.list, calendar.list_events, git.log | Generates standup update at a scheduled time, delivers as notification |
| Expense Tracker | ReAct + Co-pilot | screen.ocr, camera.capture_photo, memory.store, notes.append | Receipt scan → structured expense log entry |
| Wellness Check | Autopilot | health.steps, health.sleep, health.heart_rate | Morning briefing extension with yesterday's health data + daily suggestion |
| Reading Digest | Autopilot | readinglist.list, readinglist.read | Daily summary of unread saved articles grouped by topic |
| Git Assistant | ReAct | git.*, github.*, memory.store | Commit messages, PR descriptions, change summaries, issue triage |
| Code Reviewer | Plan-Execute | git.diff, github.get_pr, code.run, snippets.search | Multi-file PR review with findings and inline suggestions |
| Doc Helper | ReAct | docs.fetch, docs.search, snippets.store | Answer questions from locally cached documentation |
| Shell Assistant | ReAct + Co-pilot | shell.run, devenv.*, files.* | System diagnostics, process management, safe shell task execution |
| API Tester | ReAct | http.get, http.post, memory.store, snippets.store | Natural language → REST API calls with schema inference and snippet saving |

## Agent Executor Lifecycle

```dart
class AgentExecutor {
  Future<AgentSession> start(AgentConfig config, String userInput);
  Stream<AgentEvent> get events;    // tokens, tool calls, status updates
  Future<void> pause();
  Future<void> resume();
  Future<void> cancel();
}

sealed class AgentEvent {}
class TokenEvent extends AgentEvent { final String token; }
class ToolCallEvent extends AgentEvent { final ToolCall call; }
class ToolResultEvent extends AgentEvent { final ToolResult result; }
class ThinkingEvent extends AgentEvent { final String thought; }
class CompletedEvent extends AgentEvent { final String finalResponse; }
class ErrorEvent extends AgentEvent { final String error; }
```

## Inter-Agent Communication (Plan-and-Execute)

Sub-agents spawned by the planner share:
- Read access to the parent session's context (task description, previous results)
- Write access to a shared result buffer (keyed by subtask ID)
- No direct message passing between sub-agents (results flow through parent)

Sub-agents are ephemeral — they do not persist memory or history.

## Session Checkpoints & Rollback

Inspired by Cursor's checkpoint system. The agent executor automatically saves a checkpoint
**before each tool execution**. The user can roll back to any checkpoint, undoing the
conversation messages and tool effects that came after it.

```dart
class SessionCheckpoint {
  final String id;             // UUID
  final String sessionId;
  final String label;          // auto-label: "Before [tool_name]" or user-set
  final DateTime createdAt;
  final int messageCount;      // how many messages in context at this point
  final List<Message> messages; // snapshot of the full context at checkpoint time
}
```

**Checkpoint trigger points**:
- Before every `ToolCall` (automatic, unlabeled)
- On user request via `karmik.checkpoint` tool or the `/checkpoint` slash command (user-labeled)
- Before any Co-pilot action card is approved (ensures rollback is always possible after approval)

**Rollback behavior**:
- Messages after the checkpoint are removed from the ephemeral context
- The Tier 2 (SQLite) history is **not** reverted — rolled-back messages are soft-deleted
  (marked `is_rolled_back = true`), not physically deleted, preserving auditability
- Tool effects (e.g., a task that was created) are **not** automatically undone — the agent
  is re-prompted with: "Session was rolled back to before this action. If you need to undo
  the external effect, ask me and I will help."
- Checkpoints older than the session are not available (checkpoints are ephemeral, not persisted)

**UI surface**: In the chat view, each assistant turn has a `···` overflow menu with
"Roll back to here". The user sees a confirmation: "This will remove X messages from this
conversation. Continue?"

Maximum checkpoints retained per session: 50 (FIFO eviction of oldest).

## Pinned Context (Always-Include)

Each agent can have a "Pinned Context" block — free-form markdown text that is always
injected into the agent's context between the base system prompt and the first user message.
Think of it as a per-agent `CLAUDE.md` or `.cursorrules` equivalent.

```dart
class AgentConfig {
  // ... existing fields
  final String? pinnedContext;  // markdown, injected after system prompt, before history
}
```

**Use cases**:
- "Always respond in Portuguese"
- "The user's name is Jagadeesh. Their timezone is IST (+5:30)."
- "Current project: Karmik mobile app. Stack: Flutter + Dart + llama.cpp."
- Custom instructions that apply to every conversation with this agent

**Where it appears in context**:
```
[system_prompt]
[pinned_context]   ← always here, even on first message
[memory_injection] ← relevant semantic memories
[conversation_history]
[current_user_message]
```

**UI**: Editable in Agent Settings → Persona → "Pinned context" text field (below the system
prompt editor). Character limit: 2,000. Shows a live token count estimate.

**Interaction with plugins**: Plugin rules (from `specs/06-plugin-system.md`) are injected
after pinned context. The order is: `system_prompt → pinned_context → plugin_rules → history`.

## Agent Import / Export

Agents can be exported as a `.karmik-agent` file (JSON, optionally encrypted).
The export includes: config, system prompt, pinned context, plugin references (not plugin code),
tool permissions.
Memory is not exported by default (opt-in, separate export).

This enables agent sharing in the future community/marketplace.
