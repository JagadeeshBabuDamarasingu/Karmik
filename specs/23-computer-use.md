# Karmik — Computer Use & Desktop Automation

## Overview

Computer use gives agents the ability to observe and control the desktop GUI — clicking,
typing, scrolling, finding UI elements, and replaying recorded macros. Combined with
`screen.capture` and `screen.ocr` (read-only, already in the tool registry), agents can
now complete full GUI automation workflows.

**Platform**: macOS, Windows, Linux only. Not available on Android (sandboxed), iOS
(sandboxed), or Web. Desktop-only tools.

**Security posture**: all `computer.*` tool calls default to Co-pilot mode. The user sees
exactly what the agent is about to do before it happens. Batch approval is available for
recorded macros (approve once, agent replays without prompting for each step).

---

## Tools

### Core Input Tools

```
computer.screenshot
  → Take a screenshot of the current screen or a region
  Input: { "region"?: { "x": int, "y": int, "width": int, "height": int },
           "display"?: int }  // for multi-monitor setups (0 = primary)
  Output: { "imagePath": string, "width": int, "height": int }
  Note: this is a higher-detail variant of screen.capture with display selection.
        Agents should use this to "see" before clicking.
  Co-pilot: no (read-only; safe)

computer.find_element
  → Find a UI element on screen by text, image, or accessibility label
  Input: {
    "text"?: string,            // text content to match (exact or fuzzy)
    "image"?: string,           // base64 PNG template to match (template matching)
    "role"?: string,            // accessibility role: "button" | "textfield" | "checkbox" | ...
    "region"?: { x, y, w, h }, // limit search to a region
    "confidence"?: float        // match confidence threshold 0.0–1.0, default 0.8
  }
  Output: { "found": bool, "x": int, "y": int, "width": int, "height": int,
            "text"?: string, "confidence": float }
  Co-pilot: no (read-only; safe)

computer.click
  → Click, double-click, or right-click at a coordinate or on a found element
  Input: { "x"?: int, "y"?: int,           // coordinate (use if find_element result known)
           "elementText"?: string,          // shortcut: find by text then click
           "button"?: "left|right|middle",  // default "left"
           "clicks"?: int,                  // 1=single, 2=double, default 1
           "modifiers"?: string[]           // "shift" | "ctrl" | "cmd" | "alt"
         }
  Output: { "clicked": bool, "x": int, "y": int }
  Co-pilot: always

computer.type
  → Type text at the current cursor position
  Input: { "text": string, "clearFirst"?: bool, "intervalMs"?: int }
    // clearFirst: select all + delete before typing
    // intervalMs: delay between keystrokes (default 0 = as fast as possible; use ~50 for slow)
  Output: { "typed": bool, "characters": int }
  Co-pilot: always

computer.key
  → Press a keyboard shortcut or special key
  Input: { "key": string }
    // Examples: "cmd+c", "ctrl+z", "enter", "escape", "tab", "up", "f5", "cmd+shift+4"
    // Key format: modifier+modifier+key (modifiers: cmd/ctrl/alt/shift/win)
  Output: { "pressed": bool }
  Co-pilot: always

computer.scroll
  → Scroll in a direction at a coordinate or element
  Input: { "x"?: int, "y"?: int, "direction": "up|down|left|right",
           "amount"?: int }  // scroll units, default 3
  Output: { "scrolled": bool }
  Co-pilot: no (scroll is non-destructive)

computer.drag
  → Drag from one coordinate to another
  Input: { "fromX": int, "fromY": int, "toX": int, "toY": int,
           "durationMs"?: int }  // drag speed, default 500ms
  Output: { "dragged": bool }
  Co-pilot: always

computer.get_clipboard
  → Read the current clipboard content
  Input: {}
  Output: { "text"?: string, "imagePath"?: string }
  Co-pilot: no (read-only)

computer.set_clipboard
  → Write text or an image to the clipboard
  Input: { "text"?: string }
  Output: { "written": bool }
  Co-pilot: no
```

### Macro Tools

```
automation.record
  → Start recording a macro (captures all user interactions)
  Input: { "name": string, "description"?: string }
  Output: { "recordingId": string }
  Note: recording continues until automation.stop_record is called.
        All keyboard/mouse events are captured and stored locally.
  Co-pilot: always (user confirms starting a recording session)

automation.stop_record
  → Stop recording and save the macro
  Input: { "recordingId": string }
  Output: { "macroId": string, "steps": int }

automation.play
  → Replay a saved macro
  Input: { "macroId": string, "speedMultiplier"?: float, "batchApproval"?: bool }
    // batchApproval: if true, user approves the whole replay at once (not step-by-step)
    // speedMultiplier: 1.0 = original speed, 2.0 = twice as fast
  Output: { "played": bool, "steps": int, "errors": [] }
  Co-pilot: always (or batch approval if batchApproval: true)

automation.list_macros
  → List saved macros
  Input: {}
  Output: [{ "id", "name", "description", "steps": int, "createdAt": ISO8601 }]

automation.delete_macro
  → Delete a saved macro
  Input: { "id": string }
  Output: { "deleted": bool }
```

---

## Platform Implementations

### macOS

| Tool | Implementation | API |
|---|---|---|
| `computer.screenshot` | `CGWindowListCreateImageFromArray` | CoreGraphics |
| `computer.find_element` | AX (Accessibility) API (`AXUIElement`) for role/label; template matching for image | CoreFoundation |
| `computer.click` | `CGEventCreateMouseEvent` | CoreGraphics |
| `computer.type` | `CGEventKeyboardSetUnicodeString` | CoreGraphics |
| `computer.key` | `CGEventCreateKeyboardEvent` | CoreGraphics |
| `computer.scroll` | `CGEventCreateScrollWheelEvent` | CoreGraphics |
| `computer.drag` | `CGEventCreateMouseEvent` (sequence: down → move → up) | CoreGraphics |

**macOS permission**: requires **Accessibility** permission in System Settings → Privacy &
Security → Accessibility. Karmik must be added to the allowlist. On first use of any
`computer.*` tool, Karmik prompts the user and deep-links to the system settings pane.

Additionally, `computer.screenshot` requires **Screen Recording** permission.

### Windows

| Tool | Implementation | API |
|---|---|---|
| `computer.screenshot` | `BitBlt` (GDI) or `IDXGIOutputDuplication` (DXGI, higher performance) | Win32/DXGI |
| `computer.find_element` | `IUIAutomation` (UI Automation) for role/label; Win32 template matching | UIAutomation COM |
| `computer.click` | `SendInput` with `INPUT_MOUSE` | Win32 |
| `computer.type` | `SendInput` with `INPUT_KEYBOARD` (Unicode) | Win32 |
| `computer.key` | `SendInput` with virtual key codes | Win32 |
| `computer.scroll` | `SendInput` with `MOUSEEVENTF_WHEEL` | Win32 |
| `computer.drag` | `SendInput` sequence (mouse down → move → up) | Win32 |

**Windows permission**: no explicit user permission required for `SendInput` on Windows.
`IDXGIOutputDuplication` (for high-performance screen capture) requires the app to run
without being a UWP sandboxed process — the MSIX installer handles this.

### Linux

| Tool | X11 implementation | Wayland implementation |
|---|---|---|
| `computer.screenshot` | `XGetImage` or `scrot` | `grim` (wlr-screencopy protocol) |
| `computer.find_element` | AT-SPI2 (`libatspi`) | AT-SPI2 (`libatspi`) |
| `computer.click` | `xdotool click` | `ydotool click` (requires `uinput` module) |
| `computer.type` | `xdotool type` | `ydotool type` |
| `computer.key` | `xdotool key` | `ydotool key` |
| `computer.scroll` | `xdotool scroll` | `ydotool scroll` |
| `computer.drag` | `xdotool mousemove --sync` + click | `ydotool mousemove` + click |

**Linux permission**: on X11, no extra permissions needed. On Wayland, `ydotool` requires
loading the `uinput` kernel module (`sudo modprobe uinput` or add to `/etc/modules`). Karmik
detects which compositor is active and picks the appropriate backend automatically.

AT-SPI2 must be enabled: `export AT_SPI_BUS_LAUNCHER=...` (enabled by default on GNOME; may
need `sudo systemctl enable --user at-spi-dbus-bus.service` on some distros).

---

## Coordinate System

All coordinates use screen pixels. For multi-monitor setups:

- Coordinates are global (relative to the unified virtual display space)
- `display` parameter on `computer.screenshot` selects which monitor (0 = primary)
- `computer.find_element` searches across all displays by default

**HiDPI / Retina**: `computer.find_element` and `computer.screenshot` return coordinates
in logical pixels (Flutter's coordinate space). The native layer handles the DPI scaling
conversion so Dart code uses a single consistent coordinate space.

---

## Element Finder Details

`computer.find_element` uses three strategies, tried in order:

1. **Accessibility tree** (preferred): queries the OS accessibility API for elements with
   matching role and text. Exact and substring matching. Fast and reliable for standard UI.
2. **Image template matching**: OpenCV-style template matching against a provided screenshot
   snippet (PNG base64). Used for apps that don't expose good accessibility labels.
3. **OCR fallback**: takes a screenshot, runs OCR, finds the text, returns its bounding box.
   Slowest but works for any UI (games, screen-captured content, PDFs displayed in viewers).

---

## Macro Storage

Macros are stored in `karmik.db` in a `macros` table:

```
macros (id TEXT, name TEXT, description TEXT, steps_json TEXT, created_at TEXT)
```

Each `step` in `steps_json` is a recorded `computer.*` tool call with its exact input
parameters. Timestamps between steps are preserved (to replay timing accurately).

---

## Safety Model

Computer use is powerful and irreversible. The safety model:

| Risk level | Policy |
|---|---|
| Read-only (`computer.screenshot`, `computer.find_element`, `computer.scroll`) | No Co-pilot required |
| Input actions (`computer.click`, `computer.type`, `computer.key`, `computer.drag`) | Co-pilot required per action |
| Macro replay (`automation.play`) | Co-pilot once (batch approval covers entire replay) |
| Macro recording (`automation.record`) | Co-pilot to start recording |

**Agent loop limits**: in a single ReAct loop, the agent may issue at most 20 computer input
actions before requiring a user confirmation to continue. This prevents runaway automation.

**Sensitive content filter**: before executing `computer.type`, Karmik checks if the target
field is a password input (accessibility role `AXSecureTextField` on macOS, `passwordEdit` on
Windows). If so, Co-pilot shows a stern warning: "This will type into a password field."

**Audit log**: all `computer.*` and `automation.*` calls are logged to the audit log with
`"tool": "computer.click"`, `"coords": [x, y]`, `"timestamp"`, and the active agent ID.

---

## New Built-in Agent

**Desktop Automation** agent (ReAct + Co-pilot):

| Field | Value |
|---|---|
| Mode | ReAct + Co-pilot |
| Tools | `computer.*`, `automation.*`, `screen.capture`, `screen.ocr`, `files.*`, `shell.run` |
| Purpose | Automate GUI tasks: fill forms, extract data from apps, record + replay workflows |

Example flows:
- "Fill in this timesheet form using last week's data from my notes"
- "Record me doing this data entry once, then repeat it for the other 20 rows"
- "Extract the table from this PDF viewer and save it as a CSV"

---

## Example Flow

```
User: "Submit this form on the page — fill Name with my name, Email with my email"
        │
Desktop Automation (ReAct + Co-pilot):
  → computer.screenshot {}
    ✓ [screenshot showing a web form with Name and Email fields]
  → computer.find_element { "role": "textfield", "text": "Name" }
    ✓ { found: true, x: 420, y: 310 }
  → [Co-pilot card]: "Click on the Name field at (420, 310)?" [Approve] [Cancel]
    ✓ Approved
  → computer.click { "x": 420, "y": 310 }
    ✓ clicked
  → memory.search { "query": "my full name" }
    ✓ "Jagadeesh Damarasingu"
  → [Co-pilot card]: "Type 'Jagadeesh Damarasingu' into the Name field?" [Approve] [Edit] [Cancel]
    ✓ Approved
  → computer.type { "text": "Jagadeesh Damarasingu" }
    ✓ typed
  → [... repeats for Email field ...]
  Response: "Form filled. Name: Jagadeesh Damarasingu, Email: jagadeesh@… Ready to submit?"
```
