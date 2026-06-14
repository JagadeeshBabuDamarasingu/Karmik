# Karmik — Voice

## Overview

Karmik supports a full voice pipeline: speech-to-text (already spec'd in `07-tool-registry.md`
as `microphone.record` + `stt.transcribe`), text-to-speech for agent responses, push-to-talk
input, optional wake-word activation, and a hands-free voice-first mode for driving, cooking,
and workouts.

---

## Text-to-Speech (TTS)

Karmik can speak agent responses aloud. Three backends are tried in priority order:

### Backend 1 — On-Device (Preferred)

**Piper TTS** — fast, natural-sounding, privacy-safe.

- Runtime: Dart FFI → native shared library (`libpiper.so` / `piper.dll` / `libpiper.dylib`)
- Model format: `.onnx` voice model + `.onnx.json` config (~50 MB per voice)
- Languages: 16 languages, multiple voices per language (e.g., `en_US-lessac-medium`,
  `en_GB-alba-medium`, `de_DE-thorsten-medium`)
- Latency: first audio chunk < 150ms on a mid-range device (Piper is not autoregressive —
  it generates entire sentences in one ONNX forward pass)
- Works offline. Works in Privacy Mode.

Download: voice models are downloaded on demand (not bundled). First-time TTS use triggers
a voice model download prompt (~50MB). Models cached in `karmik/tts/` in app storage.

```dart
class PiperTtsEngine implements TtsEngine {
  Future<void> speak(String text, {PiperVoice? voice, double speed = 1.0});
  Future<void> stop();
  Stream<TtsWordEvent> get wordEvents;    // word-level timing for karaoke highlight
}
```

### Backend 2 — Platform TTS (Fallback)

Built-in OS speech synthesis. No download required. Lower naturalness than Piper.

| Platform | API |
|---|---|
| Android | `android.speech.tts.TextToSpeech` |
| iOS | `AVSpeechSynthesizer` |
| macOS | `NSSpeechSynthesizer` / `AVSpeechSynthesizer` |
| Windows | `Windows.Media.SpeechSynthesis.SpeechSynthesizer` |
| Linux | `espeak-ng` (called via subprocess) |
| Web | `window.speechSynthesis` (Web Speech API) |

Platform TTS is used when:
- No Piper voice model is downloaded yet (also triggers background download)
- Piper fails to initialize (corrupted model, FFI error)
- User has explicitly selected "System voice" in voice settings

### Backend 3 — Cloud TTS (Opt-In)

High quality, natural voices. Requires internet. **Blocked in Privacy Mode.**

| Provider | Notes |
|---|---|
| ElevenLabs | Highest quality; requires API key; ~0.3¢/1K chars |
| OpenAI TTS | `tts-1` / `tts-1-hd`; 6 voices; ~1.5¢/1K chars |
| Google Cloud TTS | WaveNet + Studio voices; ~1.6¢/1K chars |

Cloud TTS is disabled by default. User opts in per-agent in Agent Settings → Voice.

### TTS Engine Interface

```dart
abstract class TtsEngine {
  Future<void> speak(String text, {TtsVoice? voice, double speed = 1.0});
  Future<void> stop();
  Future<void> pause();
  Future<void> resume();
  Stream<TtsEvent> get events;
}

sealed class TtsEvent {}
class TtsStartedEvent extends TtsEvent {}
class TtsWordEvent extends TtsEvent { final String word; final Duration offset; }
class TtsSentenceEvent extends TtsEvent { final String sentence; }
class TtsFinishedEvent extends TtsEvent {}
class TtsErrorEvent extends TtsEvent { final String error; }
```

### Streaming TTS (Low-Latency Voice Response)

When TTS is enabled, Karmik does not wait for the full agent response before speaking.
Instead, it streams:

```
Model generates tokens → sentence boundary detected → Piper renders sentence → audio plays
                                                                    ↕ (overlapping)
                         Model continues generating next sentence    ↗
```

**Latency target**: < 800ms from end of user speech to first spoken word of response
(on mid-range device with Piper already loaded).

**Sentence boundary detection**: split on `.`, `!`, `?`, `\n\n` — with minimum 20-token
lookahead to avoid splitting mid-abbreviation (e.g., "e.g.", "Dr.", "vs.").

**Tool call handling**: tool call results are not spoken aloud by default. Only the
agent's final natural-language response is sent to TTS. If "Read tool calls aloud"
is enabled, tool names and summaries are read in a lower-volume voice.

---

## Push-to-Talk (PTT)

In the overlay's expanded input state, the microphone button (`🎤`) works as PTT:

```
Press and hold  → recording starts (Whisper.cpp activated, waveform shown)
Release         → recording stops → Whisper transcribes → transcript sent as message
                → agent responds (text + TTS if enabled)
```

**Keyboard shortcut** (desktop): configurable, default `Space` while overlay is focused.

**Waveform visualization**: real-time amplitude display during recording (same as existing
microphone.record UI).

**VAD (Voice Activity Detection)**: after release, Karmik waits up to 500ms for trailing
speech before finalizing. Short accidental presses (< 300ms) are ignored.

**PTT in voice-first mode**: PTT activates via proximity sensor tap or hardware button,
not screen press (screen may be off).

---

## Wake Word Detection (Always-Listening)

Optional, **off by default**. Requires explicit user opt-in and `RECORD_AUDIO` permission.

### Wake Word Engine

**openWakeWord** (primary):
- Fully on-device, open-source
- Model size: < 5 MB
- CPU usage: < 2% on mid-range Android (runs on audio thread at 16kHz, 80ms frames)
- Default wake phrase: **"Hey Karmik"**
- Custom phrase: user can replace "Hey Karmik" with any phrase (requires brief training:
  3 recording samples, takes ~10 seconds in the UI)
- Confidence threshold: configurable (default 0.7; lower = more sensitive, more false positives)

**Picovoice Porcupine** (alternative, higher accuracy, proprietary):
- Requires Picovoice API key (free tier available)
- More accurate, better false-positive rejection
- Available as an opt-in alternative in Settings → Voice → Wake Word → Engine

### Wake Word Flow

```
Background service: continuous 16kHz audio capture (Karmik foreground service)
        │
        ▼
openWakeWord frame processor (80ms frames, 16kHz, 16-bit PCM)
        │  confidence > threshold
        ▼
1. Overlay pulses (visual feedback: border animation)
2. Microphone activates for up to 8s (extended if VAD detects ongoing speech)
3. Whisper.cpp transcribes utterance
4. Transcript sent to active agent (or default agent if no active chat)
5. Agent responds; TTS speaks response if TTS is enabled
```

### Always-On Visibility

When wake word detection is active, a persistent indicator is always visible:

- **Android**: status bar notification icon (Karmik logo, colored differently from the
  background service notification). Notification text: "Hey Karmik is listening".
- **macOS**: menu bar icon shows a pulsing state (subtle, not distracting).
- **Windows**: system tray icon tooltip shows "Wake word active".
- **Linux**: status bar icon (app indicator); or a persistent floating dot overlay (fallback).

This indicator cannot be hidden. It is a privacy requirement: users must always know when
the microphone is in always-listening mode.

### Platform Availability

| Platform | Wake Word | Notes |
|---|---|---|
| Android | ✓ | Background service + NotificationListenerService; exempt from Doze |
| iOS | — | Not available. iOS prohibits always-on background mic access |
| macOS | ✓ | LaunchAgent daemon; requires microphone permission in System Settings |
| Windows | ✓ | Windows service; requires microphone permission |
| Linux | ✓ | systemd user service; requires PulseAudio/PipeWire access |
| Web | — | Browser does not permit always-on microphone access |

---

## Voice-First Mode (Hands-Free)

A dedicated UI mode designed for driving, cooking, workouts, or any scenario where
looking at the screen is inconvenient or dangerous.

### Activation

- **Manual**: overlay long-press (1.5s) → "Voice mode" button
- **Bluetooth**: auto-activate when a paired Bluetooth device (headset/car) connects
  (configured in Settings → Voice → Voice-First → Activate on Bluetooth)
- **Agent command**: agent can call `voice.mode_on` tool (with Co-pilot confirmation)

### Behavior in Voice-First Mode

- Screen stays on at 20% brightness (configurable: off / dim / full)
- All agent responses are spoken aloud by TTS (even if TTS is off in normal mode)
- User speaks to interact via wake word (if enabled) or PTT hardware button / proximity tap
- Background agent notifications are read aloud as they arrive
- Simple voice commands understood without sending to the model:
  - "Stop" / "Quiet" — stop current TTS playback
  - "Repeat" — re-read last response
  - "Louder" / "Quieter" — adjust TTS volume by 20%
  - "Faster" / "Slower" — adjust TTS speed by 0.2×
  - "Stop voice mode" / "Exit voice mode" — deactivate

### Deactivation

- Voice command: "Stop voice mode"
- Double-tap proximity sensor (if device has one)
- Bluetooth headset disconnect (if activated by Bluetooth)
- Overlay → "Exit voice mode" button (always visible in a bottom strip)

### Visual UI in Voice-First Mode

```
┌─────────────────────────────────────────────┐
│                                             │
│          ARIA                               │
│   ╔══════════════════════════════╗          │
│   ║ "Your standup update is      ║          │
│   ║  ready. You have 3 tasks     ║          │  ← Scrolling transcript
│   ║  due today..."               ║          │
│   ╚══════════════════════════════╝          │
│                                             │
│         ●  ●  ●  ●  ●  ●  ●               │  ← TTS waveform
│                                             │
│   ┌───────────────────────────────────┐    │
│   │  [Repeat]  [Stop]  [Exit voice]  │    │  ← Always-visible strip
│   └───────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

---

## Tools (for Agent Use)

Two tools allow agents to programmatically control TTS and voice mode:

```
tts.speak
  → Speak text aloud using TTS
  Input: { "text": string, "voice"?: string, "speed"?: float }
  Output: { "spoken": bool, "durationMs": int }
  Note: non-blocking by default (starts speaking, returns immediately).
        Agent can use tts.speak for proactive alerts without full response.

tts.stop
  → Stop current TTS playback immediately
  Input: {}
  Output: { "stopped": bool }

voice.mode_on
  → Activate voice-first mode (Co-pilot required)
  Input: { "reason"?: string }
  Output: { "active": bool }
  Co-pilot: always — shows user why the agent wants to switch to voice mode

voice.mode_off
  → Deactivate voice-first mode
  Input: {}
  Output: { "active": bool }
```

---

## Voice Settings (Agent Settings → Voice)

```
Voice
  Speak responses                  [✓ On / Off]
  Voice                            [Piper — Emma (en_US) ▾]
    Download more voices           [→]
  Speed                            [———●———  1.0×]
  Read tool calls aloud            [Off]
  Cloud voice (higher quality)     [Off ▾]
    Provider                       [ElevenLabs ▾]
    API key                        [•••••••••••••]

Wake Word
  "Hey Karmik"                     [Off / On]
  Custom phrase                    [Hey Karmik      [Change]]
  Sensitivity                      [—●————  Medium]
  Engine                           [openWakeWord ▾]

Voice-First Mode
  Activate on Bluetooth            [Off / On]
  Screen in voice mode             [Dim (20%) ▾]
```

**Global vs. per-agent**: TTS on/off and voice selection are per-agent. Wake word
and voice-first mode activation are global settings (they don't belong to one agent).

---

## Privacy Mode

| Feature | Privacy Mode behavior |
|---|---|
| Piper TTS (on-device) | Allowed |
| Platform TTS | Allowed |
| Cloud TTS (ElevenLabs, OpenAI, Google) | Blocked |
| Wake word detection | Allowed (on-device openWakeWord / Porcupine) |
| STT via Whisper.cpp | Allowed |
| STT via cloud API | Blocked (governed by `stt.transcribe` Privacy Mode rules) |

---

## Latency Budget (Voice-to-Voice Round Trip)

Target: **< 800ms** from end of user speech to first spoken word of TTS response.

| Step | Budget |
|---|---|
| Whisper transcription (Whisper.cpp, small model) | 200–400ms |
| Model TTFT (first token, local Phi-3.8B, Vulkan) | 150–300ms |
| First sentence generation (~15 tokens) | 50–100ms |
| Piper TTS render + audio buffer fill | 100–150ms |
| **Total** | **500–950ms** |

On high-end devices (flagship SoC + large VRAM): < 500ms achievable.
On low-end devices: falls back to platform TTS (which starts speaking earlier than Piper
since no FFI call is needed) + a smaller / quantized model.

---

## Platform Summary

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| TTS (Piper, on-device) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| TTS (platform) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| TTS (cloud) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Push-to-talk | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Wake word | ✓ | — | ✓ | ✓ | ✓ | — |
| Voice-first mode | ✓ | ~ | ✓ | ✓ | ✓ | — |
| Streaming TTS | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**iOS voice-first (~)**: voice-first mode is available but wake word is not. PTT via
on-screen button is the input method. Background TTS requires the audio session to be kept
active (uses `AVAudioSession` backgroundModes: `audio`).

**Web**: Piper TTS requires WASM build of ONNX Runtime — not shipped in v1. Platform TTS
(Web Speech API) is the only TTS on Web.
