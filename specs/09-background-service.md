# Karmik — Background Service & Orchestration Daemon

## Overview

Karmik runs a persistent Android background service that acts as an orchestration daemon.
It watches for triggers, spawns agent sessions, and delivers results — all without requiring
the user to open the app.

This is what separates Karmik from a chat app: agents can *initiate* work, not just respond.

## Android Service Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  KarmikForegroundService (Android Foreground Service)            │
│                                                                  │
│  Always running when Karmik is active.                           │
│  Shows a persistent (minimal) notification to satisfy Android    │
│  foreground service requirements.                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  TriggerEngine                                           │    │
│  │  Watches for conditions → fires trigger events           │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │ trigger fired                      │
│  ┌──────────────────────────▼──────────────────────────────┐    │
│  │  OrchestratorPool                                        │    │
│  │  Spawns, manages, and terminates agent executor sessions │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │ result                             │
│  ┌──────────────────────────▼──────────────────────────────┐    │
│  │  NotificationDelivery                                    │    │
│  │  Delivers results as Android notifications with actions  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  WorkManager (Android WorkManager)                               │
│                                                                  │
│  For deferrable, battery-friendly tasks.                         │
│  Used when exact timing is not required (daily summaries, etc.)  │
│  Respects Doze mode and battery optimization.                    │
└─────────────────────────────────────────────────────────────────┘
```

## Trigger Types

### 1. Notification Trigger

Fires when a new Android notification arrives matching a rule.

```dart
class NotificationTrigger extends Trigger {
  final String? appFilter;         // "com.whatsapp" | null = all apps
  final String? contentPattern;    // regex matched against notification body
  final List<String> agentIds;     // agents to invoke when triggered

  // Built on Android NotificationListenerService
  // Requires user to grant notification access permission
}
```

Example: "When I get a WhatsApp notification containing a task-like phrase, invoke the
Task Capture agent."

### 2. Schedule Trigger

Fires at a specific time or on a cron-like schedule.

```dart
class ScheduleTrigger extends Trigger {
  final String cronExpression;     // "0 7 * * *" = 7am daily
  final String agentId;
  final String taskDescription;    // what to tell the agent when it fires

  // Implemented via WorkManager PeriodicWorkRequest (for recurring)
  // or OneTimeWorkRequest (for one-shot)
}
```

Examples:
- Morning Briefing: `0 7 * * 1-5` (7am, weekdays)
- Weekly review: `0 9 * * 0` (9am, Sundays)
- End-of-day task review: `0 18 * * 1-5`

### 3. Location Trigger (Geofence)

Fires when the device enters or exits a defined geographic area.

```dart
class LocationTrigger extends Trigger {
  final double latitude;
  final double longitude;
  final double radiusMeters;
  final GeofenceEvent event;       // enter | exit | dwell
  final String agentId;
  final String taskDescription;

  // Implemented via Android Geofencing API
}
```

Example: "When I arrive at the office, run the Work agent to brief me on the day."

### 4. Custom Trigger (App Events)

Fires on Karmik-internal events.

```dart
class CustomTrigger extends Trigger {
  final CustomTriggerType type;
  final String agentId;
}

enum CustomTriggerType {
  onScreenUnlock,         // when user unlocks the phone
  onCharging,             // when device is plugged in
  onBatteryFull,          // good time for heavy tasks
  onAppOpened,            // when a specific app is opened
  onConversationEnd,      // after another agent finishes
}
```

### 5. Smart Home Trigger

Fires when a smart device's state matches a condition. Requires at least one
`SmartHomeAdapter` configured (see `specs/16-smart-home.md`).

```dart
class SmartHomeTrigger extends Trigger {
  final String deviceId;               // specific device to watch
  final String? attributeKey;          // "on" | "temperature" | "locked" | null (= any change)
  final TriggerCondition condition;
  final List<String> agentIds;

  // Implemented by subscribing to SmartHomeAdapter.stateChanges (WebSocket/MQTT stream)
  // Evaluated in the foreground service; no polling — purely event-driven
}

class TriggerCondition {
  final ConditionOperator operator;    // eq | neq | gt | lt | gte | lte | changed
  final dynamic value;                 // threshold or expected value
}
```

**Example triggers**:
- Front door unlocked → run Security Check agent
- Living room temperature < 18°C → run Heating agent
- Motion sensor active between 00:00–06:00 → alert immediately via `karmik.notify`
- TV turned on → start "Focus block" agent (dim lights, silence notifications)

## Orchestrator Pool

Manages concurrent agent sessions spawned by triggers.

```dart
class OrchestratorPool {
  static const int maxConcurrentSessions = 2;  // memory constraint

  Future<String> spawn(AgentConfig agent, TriggerEvent trigger);
  Future<void> terminate(String sessionId);
  Stream<OrchestratorEvent> get events;

  // If pool is full: queue the trigger, execute when a slot opens
  // Queue max size: 10 (oldest dropped if exceeded)
}
```

**Model loading strategy for background sessions**:
- If the model needed by the triggered agent is already warm (another session just used it):
  reuse the loaded model (no reload cost)
- If a different model is needed: unload current, load new one (~5–30s depending on model size)
- If device is in Doze mode: defer model load to WorkManager, use a tiny fast model as fallback

## Battery Management

| Scenario | Strategy |
|---|---|
| Active foreground use | Full model loaded, foreground service running, wake-lock held during inference |
| Screen off, plugged in | Background sessions can run freely; schedule heavy model-load tasks here via `onChargingTrigger` |
| Screen off, on battery | Defer non-critical tasks to WorkManager (battery-aware scheduler); WorkManager minimum interval is 15 minutes |
| Doze mode | Only `AlarmManager.setExactAndAllowWhileIdle()` wakes are permitted; exact-time schedule triggers use this; notification triggers fire via `NotificationListenerService` which is exempt from Doze |
| Low battery (<15%) | `BatteryManager.EXTRA_LEVEL` broadcast triggers suspension of all background agent sessions; queued triggers are preserved and resume when battery recovers above 20% |

**Doze mode detection**: The app registers for `PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED`
broadcast. When `PowerManager.isDeviceIdleMode()` returns `true`, the `OrchestratorPool`
stops spawning new sessions. In-progress sessions are allowed to complete (inference is
CPU-bound and does not require network), then the pool idles until Doze ends.

**WorkManager constraints** used for deferrable tasks (e.g., daily summary, memory compaction):
```dart
final constraints = Constraints(
  requiredNetworkType: NetworkType.not_required,
  requiresBatteryNotLow: true,
  requiresCharging: false,   // relaxed — runs on battery if not low
);
```
WorkManager's minimum repeat interval is 15 minutes (Android OS limit). Any schedule trigger
requiring finer granularity (e.g., "every 5 minutes") must use `AlarmManager` instead and
will drain battery faster — warn the user in the trigger configuration UI.

**Wake-lock strategy**: The foreground service holds a `PARTIAL_WAKE_LOCK` only during active
inference (between `ModelRuntime.infer()` call and stream completion). It is released
immediately after each inference call. This prevents screen-off from interrupting a mid-stream
response while avoiding continuous wake-lock that would drain battery.

**Foreground service notification** (minimal, persistent while background mode is active):
- Icon: Karmik logo (small)
- Text: "Karmik is watching for triggers" or "Karmik: [agent name] is working..."
- Action button: "Pause background" (suspends all triggers without uninstalling)

## Result Delivery

Results from background agents are delivered as Android notifications.

```dart
class AgentResultNotification {
  final String agentName;
  final String summary;            // short summary for notification body
  final String? fullResponse;      // full response opened in app on tap
  final List<NotificationAction> actions;  // quick-action buttons
}

class NotificationAction {
  final String label;              // "Confirm", "Reschedule", "Dismiss"
  final ActionIntent intent;       // what happens when tapped
}
```

**Notification action intents**:
- `openChat`: opens the agent's chat in the app
- `executeTask`: runs a specific tool call (e.g., confirm adding calendar event)
- `dismiss`: dismisses without action
- `snooze`: re-delivers the notification in N minutes

**Example notification** (Morning Briefing agent):
```
┌─────────────────────────────────────────────┐
│ 🌅 Morning Briefing                          │
│ You have 3 open tasks and a standup at 9am.  │
│ 45 free minutes at 8am — block for prep?     │
│                                              │
│ [Block time]  [View full]  [Dismiss]        │
└─────────────────────────────────────────────┘
```

## Trigger Configuration UI

Each agent's settings screen shows a "Triggers" section:

```
Triggers
──────────────────────────────
+ When a notification arrives
  App: WhatsApp
  Contains: (any)
  → Run Task Capture agent

+ On a schedule
  7:00 AM, weekdays
  → Run Morning Briefing agent

+ When I arrive at...
  Home (set location)
  → Run Personal agent

[Add trigger]
```

## Service Lifecycle

1. **App first launch**: service not started (user must explicitly enable background mode)
2. **User enables background mode**: `startForegroundService()` called, persists across reboots
   via `RECEIVE_BOOT_COMPLETED` intent
3. **User pauses background mode**: triggers are suspended but service stays running (faster resume)
4. **User disables background mode**: `stopService()` called, triggers cleared
5. **App update**: service is restarted automatically by the OS boot intent handler

## Permissions Required

| Permission | Purpose |
|---|---|
| `FOREGROUND_SERVICE` | Run foreground service |
| `FOREGROUND_SERVICE_DATA_SYNC` | Android 14+ foreground service type |
| `RECEIVE_BOOT_COMPLETED` | Restart service after device reboot |
| `BIND_NOTIFICATION_LISTENER_SERVICE` | Read notifications (user must grant in Settings) |
| `ACCESS_FINE_LOCATION` | Geofence triggers (user must grant) |
| `POST_NOTIFICATIONS` | Deliver result notifications (Android 13+) |
| `SCHEDULE_EXACT_ALARM` | Schedule triggers at precise times |

**Minimum SDK requirement**: API 29 (Android 10). Background processing restrictions below API 29
make the trigger daemon unreliable. Vulkan 1.1 (required for GPU-accelerated llama.cpp inference)
is not reliably available on API < 29. See `specs/12-tech-stack.md` for the full Android version
support matrix.
