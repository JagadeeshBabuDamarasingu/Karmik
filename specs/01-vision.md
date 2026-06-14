# Karmik — Vision & Product Philosophy

## Name

Karmik derives from the Sanskrit word *Karma* — the principle of agency and action. The name signals
that AI should *do* things for the user, not just answer questions. Agency is the product.

## The Problem

Cloud AI assistants are powerful but fundamentally misaligned with user interests:

- **Privacy**: Every conversation, notification, and document leaves the device.
- **Cost**: Subscriptions add up. Power users pay $80–200/month across tools.
- **Availability**: Useless on airplane mode, throttled on bad connections.
- **Control**: The model, the memory, the context — none of it is yours.

Existing local AI apps (Jan, Msty, LM Studio mobile ports) solve inference but not *agency* — they
are chat windows, not agents. They cannot watch your notifications, act in the background, or
proactively get things done.

## The Solution

Karmik is a local-first agentic AI orchestration platform for all your devices. It combines:

1. **On-device inference** — models run locally, no network required (on capable hardware)
2. **Named persistent agents** — AI personas with memory, tools, and purpose
3. **Deep system integration** — agents that can read notifications, manage tasks, access calendar (platform-specific capabilities; see `specs/15-platform-support.md`)
4. **Background orchestration** — agents that act without being explicitly invoked
5. **Plugin ecosystem** — extend capabilities with skills, rules, prompts, and scripts
6. **Privacy Mode** — a hardware-locked setting that guarantees zero external network calls from the AI layer

## Positioning Statement

> **"Karmik is the AI that works for you without working for anyone else's cloud."**

This makes the local-first architecture a user benefit, not a technical constraint. Privacy is not
a checkbox — it is the core product differentiation.

## Target User

**Primary**: Privacy-conscious power users (Android, iPhone, Mac, PC, Linux) who want AI in their
daily workflow but distrust cloud services with their personal data.

**Secondary**: Productivity-focused users who want a "getting things done" AI that requires zero
subscription fees and works offline.

**Tertiary**: Users who need to keep their data on-device due to regulatory, professional, or
personal constraints (healthcare, legal, journalism).

**Quaternary (Stage 2)**: Developers who want to embed the Karmik agent runtime into their own apps
without building the inference, memory, and orchestration layers themselves.

## The Killer Demo

**"Capture Everything, Forget Nothing"**

1. User is in a meeting. A WhatsApp message arrives: *"Hey, can you send me that article?"*
2. Karmik floating overlay pulses at the screen edge.
3. User swipes → on-device agent reads the notification: sender, intent, implied task.
4. Overlay shows: *"Add 'Send Alex the article' as a task?"* — one tap confirms.
5. Task saved locally with full context: who, when, source channel.
6. Evening: background worker reads open tasks + Calendar free slots.
7. Notification: *"You have 3 captured tasks. I blocked 9–9:30am tomorrow. Confirm?"*
8. One tap → calendar event created, entirely on-device.

**Why this wins the demo**: floating overlay is unique to Android system apps, the privacy story is
self-evident (WhatsApp content never touched a server), it works on airplane mode, and the model
required (Gemma 4 2B) fits on any phone with 6GB+ RAM today.

## Roadmap

### Stage 1 — Consumer App (Android + macOS + iOS)
- On-device inference with model manager
- Named persistent agents with memory
- Floating overlay (Android/macOS) + optional launcher mode (Android)
- Core system integrations: notifications, calendar, tasks, files (platform-specific breadth)
- Plugin system (local install)
- Background orchestration service
- Optional remote inference (Groq, Gemini, Claude, custom endpoints)
- **Privacy Mode** — lockdown setting that enforces local-only inference

### Stage 2 — All Platforms + Developer SDK
- Windows, Linux, and Web support
- Dart/Swift/Kotlin SDK: embed the Karmik agent runtime in any app
- Plugin marketplace
- Agent sharing and export
- MCP server ecosystem integration
- Cross-device agent sync (optional, encrypted, user-controlled)

## Non-Goals

- A cloud service of any kind
- A general-purpose chat app (there are plenty; Karmik is about agency, not conversation)
- A model training or fine-tuning platform
- Identical feature sets across all platforms (system integration depth varies; see `specs/15-platform-support.md`)
