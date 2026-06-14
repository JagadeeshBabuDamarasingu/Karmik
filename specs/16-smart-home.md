# Karmik — Smart Home Integration

## Overview

Karmik agents can discover, monitor, and control smart devices across all major ecosystems.
The smart home layer follows the same pattern as the model runtime abstraction: a unified
`SmartHomeAdapter` interface with per-protocol implementations. Agents talk to the adapter;
the adapter handles the protocol.

**Privacy alignment**: local-network protocols (Home Assistant, Matter, Philips Hue local API,
MQTT) are preferred and work in Privacy Mode. Cloud-API protocols (Tuya, SmartThings) are
supported but blocked by Privacy Mode (they route through the internet).

**Mobile-first**: smart home control is available on Android and iOS. macOS and desktop
platforms have the same tools available via LAN when the device is on the same network as
the hub.

---

## SmartHomeAdapter Interface

```dart
abstract class SmartHomeAdapter {
  String get id;              // "homeassistant" | "homekit" | "matter" | "hue" | "mqtt" | ...
  String get displayName;
  bool get isLocal;           // true = LAN-only, works in Privacy Mode
                              // false = cloud API, blocked by Privacy Mode

  /// Discover all devices and rooms
  Future<SmartHomeSnapshot> discover();

  /// Get a single device's current state
  Future<SmartDevice> getDevice(String deviceId);

  /// Send a command to one or more devices
  Future<ControlResult> control(SmartCommand command);

  /// Get all scenes/routines
  Future<List<SmartScene>> listScenes();

  /// Activate a scene
  Future<bool> runScene(String sceneId);

  /// Subscribe to real-time state changes (for trigger-based agent wakeups)
  Stream<DeviceStateChange> get stateChanges;
}
```

### Core Data Models

```dart
class SmartDevice {
  final String id;
  final String name;
  final DeviceType type;
  final String? roomId;
  final String? roomName;
  final bool isOn;
  final Map<String, dynamic> state;      // type-specific: brightness, temp, color, etc.
  final List<DeviceCapability> capabilities;
  final String adapterId;                // which hub this device comes from
}

enum DeviceType {
  light,            // on/off
  dimmerLight,      // on/off + brightness
  colorLight,       // on/off + brightness + color (hue/sat or color_temp)
  switch_,          // on/off
  plug,             // on/off + power monitoring (optional)
  thermostat,       // set temperature + mode (heat/cool/auto/off)
  heater,
  airConditioner,
  fan,              // on/off + speed
  doorLock,         // lock/unlock
  doorbell,
  securityCamera,
  motionSensor,
  doorSensor,       // open/closed
  windowSensor,
  temperatureSensor,
  humiditySensor,
  smokeSensor,
  waterLeakSensor,
  presenceSensor,
  mediaplayer,      // play/pause/volume/input
  tv,
  blind,            // open/close/set position
  curtain,
  garageDoor,       // open/close
  irrigationValve,
  vacuumRobot,      // start/stop/dock
  custom,
}

enum DeviceCapability {
  onOff, brightness, colorTemperature, colorRGB, fanSpeed,
  targetTemperature, currentTemperature, humidity,
  locked, openClosed, motion, presence,
  volume, mediaPlayback, inputSource,
  position,         // blinds, covers
  powerConsumption,
}

class SmartCommand {
  final List<String>? deviceIds;     // specific devices (null = use room/type filter)
  final String? roomId;              // target all devices in this room
  final DeviceType? typeFilter;      // target all devices of this type
  final String command;              // "on" | "off" | "toggle" | "set"
  final Map<String, dynamic>? value; // { "brightness": 80 } | { "temperature": 21.5 } | ...
}
```

---

## Supported Hubs & Protocols

### 1. Home Assistant (Primary — Local)

**Why primary**: open-source, self-hosted, supports 3,000+ device integrations, has a
clean REST + WebSocket API, and is the most popular self-hosted smart home platform.
Works entirely on the local network — zero cloud dependency.

```dart
class HomeAssistantAdapter implements SmartHomeAdapter {
  final String baseUrl;     // e.g. "http://192.168.1.100:8123"
  final String accessToken; // long-lived access token from HA user profile
  final bool useWebSocket;  // true = real-time state_changed events
}
```

**Discovery**: the adapter calls `GET /api/states` to list all entities. Karmik maps HA
entity domains (`light`, `switch`, `climate`, `lock`, `sensor`, ...) to its `DeviceType` enum.

**Control**: `POST /api/services/{domain}/{service}` with the entity ID and optional data.
Examples:
```
POST /api/services/light/turn_on    {"entity_id": "light.bedroom", "brightness_pct": 70}
POST /api/services/climate/set_temperature  {"entity_id": "climate.living_room", "temperature": 21}
POST /api/services/lock/unlock      {"entity_id": "lock.front_door"}
```

**Real-time events**: WebSocket `subscribe_events` for `state_changed` — Karmik subscribes
when the Home Automation agent is active and uses state changes as trigger inputs.

**Auto-discovery**: Karmik scans the local subnet for port 8123 (default HA port) on first
setup. If found, prompts the user to enter their long-lived access token. No cloud account
or Nabu Casa subscription required.

### 2. Matter (Universal — Local)

Matter is an IP-based smart home standard backed by Apple, Google, Amazon, Samsung, and
most major device manufacturers. Devices join a Matter fabric and are controllable over
Thread or Wi-Fi, entirely on-device.

```dart
class MatterAdapter implements SmartHomeAdapter {
  // Matter commissioning and control via the platform's Matter SDK
  // Android: Google Home Mobile SDK (Matter commissioning support)
  // iOS/macOS: HomeKit MatterSupport framework
}
```

**Platform integration**:
- **Android**: `com.google.android.gms.home` (Google Home Mobile SDK) handles Matter
  commissioning. Karmik uses the Matter SDK to discover and control devices in the
  user's Matter fabric without requiring a Google account (local fabric only).
- **iOS/macOS**: `HomeKit` framework natively supports Matter devices. Karmik accesses
  Matter devices via the HomeKit adapter (see below).

**What Matter provides**: on/off, brightness, color temperature, thermostat, door lock,
window covering, occupancy sensor, contact sensor, temperature/humidity sensor, fan.

**Privacy**: Matter is 100% local. No cloud. Works in Privacy Mode.

### 3. HomeKit (iOS / macOS native — Local)

On iOS and macOS, Karmik uses the `HomeKit` framework directly. This gives access to
all HomeKit-compatible devices (including Matter devices bridged through HomeKit) without
requiring any hub software.

```dart
// iOS / macOS only — implemented via platform channel
class HomeKitAdapter implements SmartHomeAdapter {
  // Wraps HMHomeManager, HMHome, HMAccessory, HMService, HMCharacteristic
  // Communicates with the HomeKit controller (Apple Home app's stored fabric)
}
```

**Permissions**: `NSHomeKitUsageDescription` in Info.plist + user must grant "Home" access.

**What HomeKit provides**: full access to any device in the user's Apple Home — lights,
locks, thermostats, sensors, cameras, speakers, TVs, fans, blinds, garage doors.

**Siri Shortcuts bridge**: Karmik agents can trigger Siri Shortcuts via `NSUserActivity`,
enabling control of devices and scenes not directly accessible via the HomeKit API.

### 4. Philips Hue (Local REST API)

Direct local API — no cloud account required after initial setup.

```dart
class PhilipsHueAdapter implements SmartHomeAdapter {
  final String bridgeIp;    // e.g. "192.168.1.50"
  final String applicationKey; // obtained by pressing the bridge button during setup
  // Uses Hue API v2 (HTTPS with self-signed cert, port 443)
}
```

**Auto-discovery**: mDNS (`_hue._tcp.local`) to find Hue bridge on the local network.
Setup flow: prompt user to press the bridge button → Karmik calls `POST /api` to get an
application key → stored in the platform keystore.

**Supports**: lights (on/off/brightness/color/color_temp), groups, scenes, motion sensors,
outdoor sensors, plugs, Hue Play gradient lights.

**Privacy**: 100% local. Works in Privacy Mode.

### 5. MQTT (Generic IoT — Local or Remote)

MQTT is a lightweight pub/sub protocol used by many DIY smart home setups (ESPHome,
Tasmota, Zigbee2MQTT, openHAB). Karmik includes an MQTT adapter that lets agents
publish commands and subscribe to device state topics.

```dart
class MqttAdapter implements SmartHomeAdapter {
  final String brokerUrl;   // e.g. "mqtt://192.168.1.100:1883"
  final String? username;
  final String? password;
  final MqttDeviceMap deviceMap; // user-defined: device name → topic mapping
}
```

**Device map** (configured by the user):
```json
{
  "devices": [
    {
      "id": "living_room_light",
      "name": "Living Room Light",
      "type": "dimmerLight",
      "commandTopic": "home/living_room/light/set",
      "stateTopic": "home/living_room/light/state",
      "onPayload": "{\"state\": \"ON\"}",
      "offPayload": "{\"state\": \"OFF\"}",
      "brightnessCommandPath": "brightness"
    }
  ]
}
```

**Privacy**: local broker = works in Privacy Mode. Cloud MQTT broker (e.g., HiveMQ Cloud)
= blocked by Privacy Mode.

### 6. Shelly (Local REST API)

Shelly devices expose a local REST API (`http://{device-ip}/rpc/Switch.Set`). Karmik
discovers Shelly devices via mDNS (`_shelly._tcp.local`) and controls them directly.
No app, no cloud account, no hub required.

```dart
class ShellyAdapter implements SmartHomeAdapter {
  final List<String> deviceIps; // auto-discovered via mDNS
}
```

**Privacy**: 100% local. Works in Privacy Mode.

### 7. Tuya (Cloud — opt-in)

Tuya is the backend for thousands of budget smart home brands (Gosund, Treatlife, Kasa Pro,
etc.). Control requires a Tuya Cloud account and API credentials.

```dart
class TuyaAdapter implements SmartHomeAdapter {
  final String clientId;
  final String clientSecret;
  final String region;  // "us" | "eu" | "cn" | "in"
  // Uses Tuya IoT Cloud API (HTTPS)
}
```

**Privacy**: cloud-dependent. **Blocked by Privacy Mode.** Requires explicit user
acknowledgment: "Tuya Cloud will process device commands on their servers."

### 8. SmartThings (Cloud — opt-in)

Samsung SmartThings supports a wide range of devices (Samsung appliances, Z-Wave, Zigbee,
LAN devices) via the SmartThings Cloud API.

```dart
class SmartThingsAdapter implements SmartHomeAdapter {
  final String personalAccessToken;  // from account.smartthings.com
  // Uses SmartThings API (https://api.smartthings.com/v1)
}
```

**Privacy**: cloud-dependent. **Blocked by Privacy Mode.**

### 9. Custom HTTP (Power User)

For any smart home device that exposes a REST or local API not covered above. The user
defines:
- A list endpoint (GET → parse JSON for device list)
- A state endpoint (GET → parse JSON for device state)
- A command endpoint (POST → send command)

Useful for custom firmware, self-hosted home automation servers, or niche devices.

---

## Setup & Discovery Flow

```
Settings → Smart Home → Add hub

┌──────────────────────────────────────┐
│  Choose your smart home hub          │
│                                      │
│  [Auto-detect on local network]      │
│                                      │
│  ─ Local (Privacy-safe)              │
│  ○ Home Assistant  (detected ✓)      │
│  ○ Philips Hue     (not found)       │
│  ○ Shelly devices  (2 found)         │
│  ○ Matter          (2 devices)       │
│  ○ MQTT            (manual setup)    │
│  ○ HomeKit         (iOS/macOS only)  │
│                                      │
│  ─ Cloud (requires internet)         │
│  ○ Tuya                              │
│  ○ SmartThings                       │
│  ○ Custom HTTP endpoint              │
└──────────────────────────────────────┘
```

**Auto-detection** scans the local subnet for:
- Port 8123 (Home Assistant)
- mDNS `_hue._tcp.local` (Philips Hue)
- mDNS `_shelly._tcp.local` (Shelly)
- mDNS `_matter._tcp.local` (Matter)
- Port 1883 (MQTT broker — only if user requests it, as scanning MQTT is not always safe)

Multiple hubs can be active simultaneously. Devices from all active hubs appear in a unified
device list. The adapter ID is stored on each `SmartDevice` so commands are routed correctly.

---

## Device Permission Model

Smart home tools follow the same permission model as other Karmik tools.

**Default**: no smart home tools granted to any agent.

**Recommended per-agent grants**:

| Agent | Suggested tools |
|---|---|
| Home Automation | `smarthome.*` (full access) |
| Morning Briefing | `smarthome.query_sensor`, `smarthome.list_devices` (read-only) |
| Task Capture | `smarthome.run_scene` (scenes only, no direct device control) |
| Custom | User-configured |

**Dangerous commands** that always require Co-pilot confirmation regardless of agent mode:
- `doorLock.unlock` (unlocking a physical door)
- `garageDoor.open`
- `securityCamera.*` (accessing camera feed)

These are marked `requiresCopilot: true` in the tool registry and the `PermissionChecker`
enforces Co-pilot mode even for agents running in ReAct or Autopilot mode.

---

## Agent-Smart Home Interaction Examples

### "Goodnight" routine

```
User: "Goodnight"
        │
Agent (ReAct mode):
  → smarthome.list_devices { onlyOn: true }
    ✓ Found: living_room_light (on, 80%), kitchen_light (on), tv (on)

  → smarthome.control { room: "living_room", command: "off" }
    ✓ Turned off: living_room_light

  → smarthome.control { name: "kitchen_light", command: "off" }
    ✓ Turned off: kitchen_light

  → smarthome.control { name: "tv", command: "off" }
    ✓ Turned off: tv

  → smarthome.control { type: "thermostat", command: "set",
                        value: { temperature: 19, mode: "heat" } }
    ✓ Set thermostat to 19°C (heat mode)

  Final response: "Goodnight. I've turned off the lights and TV,
                   and set the thermostat to 19°C."
```

### Geofence trigger: "I left home"

```
LocationTrigger fires (user exits home geofence)
        │
Home Automation agent (Autopilot mode):
  → smarthome.list_devices { onlyOn: true }
    ✓ Found: porch_light (on), hallway_light (on), plug_kettle (on)

  → smarthome.control { command: "off", id: ["porch_light", "hallway_light"] }
  → smarthome.control { name: "plug_kettle", command: "off" }

  → karmik.notify { title: "Left home", body: "Turned off 2 lights and the kettle." }
```

### Sensor-based alert

```
DeviceStateChange event: motion_sensor_front_door → detected
        │
Home Automation agent (Autopilot mode, triggered by state change):
  → smarthome.query_sensor { name: "motion_sensor_front_door" }
    ✓ Motion detected at 2:14 AM

  → smarthome.get_device { type: "securityCamera" }
    [blocked — requires Co-pilot confirmation]

  → karmik.notify {
      title: "Motion at front door",
      body: "Motion detected at 2:14 AM. View camera?",
      actions: [{ label: "View", intent: "openCamera" }, { label: "Dismiss" }]
    }
```

### Natural language control

```
User (via overlay): "Dim the bedroom lights to 30% and put on the reading scene"
        │
Agent:
  → smarthome.control { room: "bedroom", type: "dimmerLight",
                        command: "set", value: { brightness: 30 } }
    ✓ bedroom_ceiling (30%), bedroom_lamp (30%)

  → smarthome.run_scene { name: "reading" }
    ✓ Scene "Reading" activated (adjusts color temperature to warm white)

  Response: "Done. Bedroom lights dimmed to 30% and Reading scene activated."
```

---

## Smart Home Trigger Type

A new trigger type is added to the background service (`specs/09-background-service.md`):

```dart
class SmartHomeTrigger extends Trigger {
  final String deviceId;
  final String? attributeKey;    // which state attribute to watch ("on", "temperature", etc.)
  final TriggerCondition condition;
  final String agentId;
  final String taskDescription;
}

class TriggerCondition {
  final ConditionOperator operator;  // eq | neq | gt | lt | gte | lte | changed
  final dynamic value;               // the threshold or expected value
}

enum ConditionOperator { eq, neq, gt, lt, gte, lte, changed }
```

Examples:
- "When front door lock changes to unlocked → run Security agent"
- "When living room temperature drops below 18°C → run Heating agent"
- "When motion sensor detects motion between midnight and 6am → alert me"

Smart home triggers subscribe to `SmartHomeAdapter.stateChanges` (WebSocket / MQTT event
stream). When the trigger condition is met, the `OrchestratorPool` spawns the configured
agent session.

---

## Privacy & Privacy Mode Behavior

| Hub | isLocal | Works in Privacy Mode |
|---|---|---|
| Home Assistant (local) | Yes | ✓ |
| Matter | Yes | ✓ |
| HomeKit (iOS/macOS) | Yes | ✓ |
| Philips Hue (local API) | Yes | ✓ |
| Shelly (local REST) | Yes | ✓ |
| MQTT (local broker) | Yes | ✓ |
| MQTT (cloud broker) | No | ✗ |
| Tuya | No | ✗ |
| SmartThings | No | ✗ |
| Custom HTTP (LAN IP) | Inferred¹ | ✓ if LAN IP |
| Custom HTTP (public URL) | No | ✗ |

¹ Karmik infers local vs. remote by checking if the hostname is an RFC 1918 address or
resolves to one. If not determinable, assumes remote (safer default).

In **LAN-only mode**: all local hubs work; cloud hubs are blocked. Karmik uses the same
RFC 1918 check to decide.

---

## UI: Smart Home Settings Screen

Accessible from Settings → Smart Home (new section after Providers).

```
Smart Home
──────────────────────────────────────────────
Connected hubs (2)
  ✓ Home Assistant     192.168.1.100:8123   [Test]  [Remove]
  ✓ Philips Hue        192.168.1.50         [Test]  [Remove]

Devices (24 total)
  Living Room  ·  6 devices
  Bedroom      ·  4 devices
  Kitchen      ·  5 devices
  ...
  [Browse all devices]

Scenes (8)
  Good Morning  ·  Good Night  ·  Movie Time  ·  ...
  [Browse all scenes]

[+ Add hub]
```

**Device browser**: shows all devices with their live state (on/off, temp, brightness).
Tapping a device shows its full state and a manual control panel. The user can test
commands directly without involving an agent.

**Live state**: the Smart Home screen refreshes device states every 30 seconds (poll)
or in real-time when a WebSocket/MQTT connection is active.
