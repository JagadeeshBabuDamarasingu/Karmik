# Karmik — System Architecture

## Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        UI Layer (Flutter)                        │
│                                                                  │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  ┌────────┐  │
│  │ Floating     │  │ Chat UI  │  │ Agent Manager│  │Launcher│  │
│  │ Overlay      │  │ (per     │  │ (create,     │  │ Mode   │  │
│  │ (bubble +    │  │  agent)  │  │  configure,  │  │(home   │  │
│  │  drawer)     │  │          │  │  monitor)    │  │ screen)│  │
│  └──────────────┘  └──────────┘  └──────────────┘  └────────┘  │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐              │
│  │ Model Manager│  │ Plugin   │  │ Memory       │              │
│  │ (browse,     │  │ Manager  │  │ Browser      │              │
│  │  download,   │  │          │  │              │              │
│  │  swap)       │  │          │  │              │              │
│  └──────────────┘  └──────────┘  └──────────────┘              │
└────────────────────────────┬────────────────────────────────────┘
                             │ Dart
┌────────────────────────────▼────────────────────────────────────┐
│                    Orchestration Engine                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Agent Executor                                          │    │
│  │  ┌──────────┐ ┌──────────────┐ ┌──────────┐ ┌───────┐  │    │
│  │  │  ReAct   │ │Plan-Execute  │ │Autopilot │ │Co-pilot│  │    │
│  │  │  Loop    │ │(planner +    │ │(background│ │(step  │  │    │
│  │  │          │ │ sub-agents)  │ │  daemon) │ │approval│  │    │
│  │  └──────────┘ └──────────────┘ └──────────┘ └───────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────┐   ┌─────────────────────────────────┐   │
│  │  Agent Registry    │   │  Plugin Runtime                  │   │
│  │  (named agents,    │   │  (sandboxed Dart isolates,       │   │
│  │   personas,        │   │   skill/rule/script execution)   │   │
│  │   configs)         │   │                                  │   │
│  └────────────────────┘   └─────────────────────────────────┘   │
└─────────┬────────────────────────┬───────────────────────────---┘
          │                        │
┌─────────▼──────────┐  ┌──────────▼──────────────────────────────┐
│  Model Runtime     │  │  Tool Registry                           │
│  Abstraction Layer │  │                                          │
│                    │  │  ┌──────────────┐  ┌───────────────────┐ │
│  ┌──────────────┐  │  │  │ Android      │  │ Network / HTTP    │ │
│  │ llama.cpp    │  │  │  │ System Tools │  │ Tools             │ │
│  │ (GGUF, FFI)  │  │  │  │ (notif, cal, │  │ (fetch, REST)     │ │
│  └──────────────┘  │  │  │  sms, files) │  └───────────────────┘ │
│  ┌──────────────┐  │  │  └──────────────┘  ┌───────────────────┐ │
│  │ MLC-LLM      │  │  │  ┌──────────────┐  │ MCP Servers       │ │
│  │ (Android)    │  │  │  │ Media Tools  │  │ (external tools   │ │
│  └──────────────┘  │  │  │ (camera, mic,│  │  via MCP protocol)│ │
│  ┌──────────────┐  │  │  │  whisper STT)│  └───────────────────┘ │
│  │ ONNX Runtime │  │  │  └──────────────┘                        │
│  └──────────────┘  │  └─────────────────────────────────────────┘
│  ┌──────────────┐  │
│  │ Cloud        │  │
│  │ Adapters     │  │
│  │ (OpenAI-compat│  │
│  │  Gemini,     │  │
│  │  Anthropic)  │  │
│  └──────────────┘  │
└────────────────────┘
          │
┌─────────▼───────────────────────────────────────────────────────┐
│  Persistence Layer                                               │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │ SQLite         │  │ sqlite-vec     │  │ Encrypted File     │  │
│  │ (conversations,│  │ (vector store, │  │ Store              │  │
│  │  tasks, agents,│  │  semantic      │  │ (model weights,    │  │
│  │  audit log)    │  │  memory)       │  │  plugin assets)    │  │
│  └────────────────┘  └────────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │
┌─────────▼───────────────────────────────────────────────────────┐
│  Android Background Service (Foreground Service + WorkManager)   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Trigger Engine                                           │    │
│  │  notification | schedule | location | geofence | custom  │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow: Full Agent Request Lifecycle

```
User input (voice/text/notification trigger)
        │
        ▼
[UI Layer] → serialize to AgentRequest
        │
        ▼
[Orchestration Engine] → load agent config (persona, system prompt, tools, memory_config)
        │
        ├─ [Memory Layer] → retrieve relevant long-term memories (vector search)
        │                 → load recent conversation history (SQLite)
        │
        ├─ [Plugin Runtime] → apply active rules to system prompt
        │
        ▼
[Agent Executor] → build context window (system + history + retrieved memories + user input)
        │
        ▼
[Model Runtime] → infer (local) or [Provider Adapter] → infer (remote)
        │
        ▼
[Agent Executor] → parse model output
        │
        ├─ [Text response] → stream to UI
        │
        └─ [Tool call] → [Tool Registry] → execute tool
                │
                ▼
        [Tool result] → append to context → loop back to model
                │
                ▼
        [Final response] → stream to UI
                │
        [Memory Layer] → persist conversation turn (SQLite)
                       → update long-term memory if significant (vector store)
                │
        [Audit Log] → append: agent, tools used, timestamp, data accessed
```

## Technology Stack Summary

| Layer | Technology |
|---|---|
| UI | Flutter (Dart) |
| Local inference | llama.cpp via Dart FFI |
| Alternate runtimes | MLC-LLM (Android), ONNX Runtime |
| Android system access | Kotlin via Flutter platform channels |
| Persistent storage | SQLite (via sqlite_async) + SQLCipher (encryption) |
| Vector search | sqlite-vec SQLite extension |
| Speech-to-text | Whisper.cpp via Dart FFI |
| Background tasks | Android WorkManager + Foreground Service |
| Plugin scripts | Sandboxed Dart isolates |
| Monorepo | Turbo + pnpm |

## Key Design Principles

1. **Local by default** — every code path works without network. Remote is opt-in, per session.
2. **Abstracted inference** — the orchestration engine never calls llama.cpp directly; it talks to `ModelRuntime`. Swap backends without touching agent code.
3. **Tool as first-class citizen** — tools are not an afterthought. The entire agent system is designed around tool use. Every capability is a tool.
4. **Audit everything** — every tool invocation is logged to the audit log before execution. The user can always see what Karmik did and why.
5. **Plugin isolation** — plugin scripts run in sandboxed Dart isolates with no ambient access to system tools. They declare their needs in the manifest.

## Module Boundaries

```
karmik/
├── apps/
│   └── mobile/          # Flutter app (UI layer only)
│       ├── lib/
│       │   ├── ui/      # Screens, widgets, overlay
│       │   └── app/     # App shell, routing, DI
├── packages/            # (Stage 2 — Dart packages)
│   ├── karmik_core/     # Orchestration engine, agent system, plugin runtime
│   ├── karmik_runtime/  # Model runtime abstraction + llama.cpp/ONNX adapters
│   ├── karmik_tools/    # Tool registry + built-in Android tools
│   ├── karmik_memory/   # Memory layer (SQLite + vector store)
│   └── karmik_providers/# Remote provider adapters
```

Stage 1 ships everything as a monolithic Flutter app. Stage 2 extracts `packages/` into a
publishable SDK.
