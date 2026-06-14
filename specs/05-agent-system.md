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
| Task Capture | ReAct | notifications.read, tasks.create | GTD capture from notifications/voice |
| Morning Briefing | Autopilot | calendar.read, tasks.read, weather | Daily summary on schedule |
| Meeting Notes | ReAct | microphone.record, tasks.create, calendar.update | Call recording → structured notes |
| Research | Plan-Execute | http.get, browser.fetch, memory.store | Deep research with source tracking |
| Home Automation | ReAct + Co-pilot | http.post, notifications.read | Smart home via API calls |

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

## Agent Import / Export

Agents can be exported as a `.karmik-agent` file (JSON, optionally encrypted).
The export includes: config, system prompt, plugin references (not plugin code), tool permissions.
Memory is not exported by default (opt-in, separate export).

This enables agent sharing in the future community/marketplace.
