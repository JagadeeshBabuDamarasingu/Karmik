# Karmik — Onboarding (First-Run Experience)

## Overview

Onboarding runs once, on first launch after install. Its goals:
1. Get the user from install to a working agent in under 5 minutes
2. Acquire the permissions Karmik needs, with clear rationale for each
3. Download a model matched to the device's capabilities
4. Make the floating overlay feel immediately useful

The onboarding sequence is a fullscreen wizard (no bottom nav, no back button). It is skipped
if the app detects it has already been completed (flag stored in `karmik_models.db`).

---

## Screen 1: Welcome

```
┌────────────────────────────────────────────┐
│                                            │
│              ◆ Karmik                      │
│                                            │
│    AI that works for you                   │
│    without working for anyone else's cloud │
│                                            │
│    • On-device intelligence                │
│    • Reads your notifications (on-device)  │
│    • Acts in the background for you        │
│    • Zero data sent to any server          │
│      (unless you choose remote models)     │
│                                            │
│              [Get started →]               │
│                                            │
└────────────────────────────────────────────┘
```

No sign-up, no account, no email. Tapping "Get started" begins the permission flow.

---

## Screen 2: Permission Flow

Permissions are requested in this order. Each gets its own explanation card before the OS
dialog appears.

### Step 2a — Notifications (POST_NOTIFICATIONS, Android 13+)

```
┌────────────────────────────────────────────┐
│  🔔 Deliver agent results                  │
│                                            │
│  Karmik uses notifications to tell you     │
│  when a background agent has finished a    │
│  task, like scheduling your meetings or    │
│  capturing a to-do.                        │
│                                            │
│  [Allow notifications →]                   │
│  [Skip for now]                            │
└────────────────────────────────────────────┘
```

If denied: "Grant later" path — onboarding continues but background agent results won't
be surfaced until granted. A banner reminder is shown on the Home screen.

### Step 2b — Overlay permission (SYSTEM_ALERT_WINDOW)

```
┌────────────────────────────────────────────┐
│  💬 The floating bubble                    │
│                                            │
│  Karmik's signature feature: a small       │
│  bubble at the screen edge, accessible     │
│  from any app. No need to switch apps to   │
│  ask your AI a question.                   │
│                                            │
│  Android requires you to grant this        │
│  manually in Settings. We'll take you      │
│  there now.                                │
│                                            │
│  [Open Settings →]                         │
│  [Skip — I'll do this later]               │
└────────────────────────────────────────────┘
```

Tapping "Open Settings" deep-links to `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`. The app
polls `Settings.canDrawOverlays()` every 500ms after the user returns from Settings. If denied,
onboarding continues (overlay features are simply hidden until granted).

### Step 2c — Notification Listener (optional at this stage)

This permission requires a trip to Settings → Apps → Special app access → Notification access.
It's presented as optional during onboarding with a strong recommendation.

```
┌────────────────────────────────────────────┐
│  👀 Read notifications (optional)          │
│                                            │
│  This lets Karmik agents see your          │
│  notifications — to capture tasks from     │
│  WhatsApp, flag calendar conflicts, etc.   │
│                                            │
│  All processing is on-device.              │
│  Karmik never sends notification content   │
│  to any server.                            │
│                                            │
│  [Grant access →]   [Skip for now]         │
└────────────────────────────────────────────┘
```

Skipped permissions are collectable later from Settings → Background → Permissions.

---

## Screen 3: Model Selection

Device capability check runs in the background during Screens 1–2. By this screen, the
recommended model is already determined.

```
┌────────────────────────────────────────────┐
│  🧠 Choose your AI model                  │
│                                            │
│  ✦ Recommended for this device:           │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ Gemma 4 2B  ·  Q4_K_M               │  │
│  │ Fast · Tool use · 1.5 GB download    │  │
│  │ Works on: 4 GB RAM devices           │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  Other options:                            │
│  ○ Phi-4-mini Q4_K_M  ·  2.3 GB          │
│  ○ Qwen 2.5 7B Q4_K_M  ·  4.5 GB  ⚠ 8GB │
│  ○ Use a remote provider (Groq, etc.)     │
│                                            │
│       [Download & continue →]             │
└────────────────────────────────────────────┘
```

- Models that fail `DeviceCapabilityChecker` are shown greyed out with a reason (e.g., "⚠ Requires 8 GB RAM")
- "Use a remote provider" skips download and goes to Step 3b (provider API key entry)
- Recommended model is pre-selected; user can change before confirming

### Step 3b — Remote provider setup (if chosen)

Quick form: select provider (Groq, Gemini, Anthropic, OpenRouter, custom), paste API key,
tap "Test connection". On success, onboarding continues. API key saved to Android Keystore.

---

## Screen 4: Download Progress

```
┌────────────────────────────────────────────┐
│  ⬇ Downloading Gemma 4 2B                 │
│                                            │
│  ████████████░░░░░░░░░░░  57%             │
│  862 MB / 1.5 GB  ·  ETA 2 min            │
│                                            │
│  [Continue in background →]               │
│                                            │
│  ─────────────────────────────────────    │
│  While you wait, meet your first agent →  │
└────────────────────────────────────────────┘
```

"Continue in background" exits the wizard to the Home screen while download continues via
the foreground service. A pinned notification tracks progress and returns the user to
onboarding when download completes.

**Low storage guard**: If `availableDiskMb < diskSpaceMb + 500` (500 MB safety margin),
show a warning before download starts:
```
⚠ Only 1.2 GB free. This model needs 1.5 GB.
Free up space or choose a smaller model.
[Choose different model]  [Continue anyway]
```

**Download interrupted** (network drop, app kill): On next launch, if onboarding flag is
not set and a `.gguf.part` file exists, the download screen is shown first with a "Resume"
button rather than restarting from scratch.

---

## Screen 5: Default Agent Setup

```
┌────────────────────────────────────────────┐
│  🤖 Meet Task Capture                      │
│                                            │
│  Your first agent is already set up.       │
│  It watches your notifications and         │
│  captures to-dos automatically.            │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ Name      Task Capture               │  │
│  │ Mode      ReAct                      │  │
│  │ Tools     notifications · tasks      │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  Want to name it something else?           │
│  [________________]                        │
│                                            │
│  [Looks good →]                            │
└────────────────────────────────────────────┘
```

The Task Capture agent is pre-built. The only customization offered here is renaming it.
Full configuration is available later in the Agents tab.

---

## Screen 6: Overlay Tutorial

Shown only if SYSTEM_ALERT_WINDOW was granted (either in Step 2b or detected as already
granted on the device).

```
┌────────────────────────────────────────────┐
│  The bubble is live  ●                     │
│                               ●            │
│  Try these gestures:                       │
│                                            │
│  👆 Tap      → Open chat                  │
│  👈 Swipe    → Dismiss / see result        │
│  👇 Long press → Switch agent              │
│                                            │
│  The bubble stays on top of every app.    │
│  Drag it to any screen edge.              │
│                                            │
│              [Finish setup ✓]              │
└────────────────────────────────────────────┘
```

The actual bubble is rendered live on-screen during this step so the user can interact with
it immediately.

---

## Screen 7: Done

```
┌────────────────────────────────────────────┐
│              ✓ You're set up               │
│                                            │
│  Karmik is ready.                          │
│                                            │
│  Next: send your agent a message or        │
│  wait for it to capture its first task.   │
│                                            │
│  [Open Karmik →]                           │
└────────────────────────────────────────────┘
```

Tapping "Open Karmik" navigates to the Home tab. Onboarding completion flag is written to
`karmik_models.db`. The screen is never shown again.

---

## Onboarding Flow Diagram

```
Launch
  │
  ▼
Onboarding complete? ──yes──► Home screen (skip onboarding)
  │ no
  ▼
Welcome screen
  │
  ▼
Permission flow
  ├── POST_NOTIFICATIONS (required for results, skippable)
  ├── SYSTEM_ALERT_WINDOW (required for overlay, skippable)
  └── Notification listener (optional)
  │
  ▼
Model selection
  ├── Local model selected ──► Download screen ──► Agent setup
  └── Remote provider selected ──► API key entry ──► Agent setup
  │
  ▼
Agent setup (Task Capture rename)
  │
  ▼
Overlay tutorial (if SYSTEM_ALERT_WINDOW granted)
  │
  ▼
Done screen → Home tab
```

---

## Re-entry Points

| Situation | Entry point |
|---|---|
| Download was backgrounded | Return to download screen on next launch if download is still in progress |
| Permission denied during onboarding | Settings → Background → Permissions shows pending grants |
| User clears app data | Onboarding re-runs from start (models re-downloaded) |
| Onboarding interrupted (app killed mid-wizard) | Resumes from last completed step (step index persisted in `karmik_models.db`) |
