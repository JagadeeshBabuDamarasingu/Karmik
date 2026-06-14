# Karmik — Accessibility & Localization

## Overview

This spec covers two related concerns:

1. **Accessibility**: making Karmik usable for people with visual, motor, auditory, and
   cognitive differences — across all 6 supported platforms.
2. **Localization (i18n)**: making Karmik available in multiple languages, with correct
   RTL layout, locale-aware formatting, and a community translation workflow.

---

## Accessibility

### Principles

Karmik targets **WCAG 2.1 Level AA** compliance. Requirements are:
- All interactive elements are reachable and operable without a pointer device
- All content is perceivable without vision
- All interactions are possible without fine motor control
- Layout adapts to user font size preferences

### Screen Reader Support

**Android (TalkBack)** and **iOS (VoiceOver)**:

- All interactive elements have semantic labels:
  ```dart
  Semantics(
    label: 'Send message',
    hint: 'Activates when you have typed a message',
    child: SendButton(),
  )
  ```
- All images and icons have `excludeSemantics: false` with meaningful descriptions
- The overlay bubble announces its state: "Karmik bubble, collapsed. Double-tap to expand."
- Agent names and tool call types are announced as actions: "Git Assistant called git.diff"

**Streaming text** (the agent typing out its response):
- Word-by-word announcement is disorienting. Instead, Karmik announces in **sentence chunks**:
  each time a sentence boundary is reached (`.`, `!`, `?`, `\n\n`), TalkBack/VoiceOver
  announces the complete sentence.
- Implementation: wrap the streaming `Text` widget in a `Semantics` widget with
  `liveRegion: true`; update the `label` property only at sentence boundaries.

**macOS (VoiceOver)** and **Windows (Narrator / NVDA / JAWS)**:
- Flutter's semantic tree is exposed to platform accessibility APIs automatically
- Custom focus order: tab navigation follows logical reading order
- All dialogs trap focus (focus does not escape outside the modal)

**Linux**: Flutter uses AT-SPI2. All semantic labels are exposed via D-Bus.

### Keyboard Navigation

Full keyboard operation on all desktop platforms (no mouse required):

| Action | macOS | Windows/Linux |
|---|---|---|
| Open overlay | ⌥Space | Ctrl+Shift+K |
| Focus chat input | Tab (when overlay open) | Tab |
| Send message | ↩ Enter | Enter |
| New line in input | ⇧↩ Shift+Enter | Shift+Enter |
| Open @mention picker | @ key | @ key |
| Open slash command picker | / key | / key |
| Navigate list items | ↑ ↓ | ↑ ↓ |
| Close overlay / dismiss | Esc | Esc |
| Approve Co-pilot action | ↩ Enter | Enter |
| Reject Co-pilot action | Esc | Esc |
| Open settings | ⌘, | Ctrl+, |
| Global search | ⌘K | Ctrl+K |
| Switch agent | ⌘⌥↑↓ | Ctrl+Alt+↑↓ |

Tab order in the main chat view: input box → send button → attachment button → voice button
→ conversation list → agent switcher → settings.

### Haptic Feedback

On devices with haptic actuators (Android, iOS, some Macs with Force Touch):

| Event | Haptic pattern |
|---|---|
| Tool call starts | Light single pulse |
| Tool call completes (success) | Medium single pulse |
| Tool call fails | Double pulse |
| Co-pilot confirmation required | Pattern: pulse-pause-pulse (attention signal) |
| Message sent | Light tick |
| Agent response arrives | Medium tick |
| Error (blocking) | Three short pulses |
| Wake word detected | Two medium pulses |

Implementation: `HapticFeedback.lightImpact()`, `mediumImpact()`, `heavyImpact()`, and
custom patterns via `HapticFeedback.vibrate()` with platform channels on Android
(`VibrationEffect.createWaveform`).

### Font Scaling

Karmik fully supports system font size from **85%** to **200%**:

- All font sizes use `sp` units (Flutter `Theme.of(context).textTheme`, which respects
  `MediaQuery.textScaleFactor`)
- No absolute pixel font sizes anywhere in the codebase (enforced by a lint rule)
- At 200% scale, the overlay expands vertically to accommodate larger text
- Long labels truncate with `TextOverflow.ellipsis`; no text is ever clipped

Minimum font size: 12sp (per WCAG non-text contrast). Maximum line length in the chat
view: 80 characters at default scale (wider at smaller font sizes — tracked via
`LayoutBuilder`).

### High Contrast Mode

Karmik detects system high contrast preference:
- Android: `MediaQuery.highContrastOf(context)`
- iOS: `MediaQuery.highContrastOf(context)` (maps to `UIAccessibility.isDarkerSystemColorsEnabled`)
- Windows: `SystemTheme.highContrast`
- macOS: `NSWorkspace.accessibilityDisplayShouldIncreaseContrast`

In high contrast mode:
- All colors switch to a high-contrast token set (4.5:1 minimum contrast ratio for text,
  3:1 for UI components — WCAG AA)
- No semi-transparent overlays (replaced with solid backgrounds)
- Focus rings use a 2px solid border in a contrasting color (not just the default ring)

**Color-blind safe palette**: the default Karmik palette is verified against deuteranopia
(red-green) and protanopia. Status colors (success/error/warning) use shape + icon in
addition to color, so no information is conveyed by color alone.

### Reduce Motion

When the user has enabled "Reduce Motion" (iOS) or "Reduce animations" (Android developer
options / Windows Ease of Access):

```dart
final bool reduceMotion = MediaQuery.disableAnimationsOf(context);
```

- If `true`: all `AnimationController` durations are set to 0
- The overlay bubble opens instantly (no scale-in animation)
- The chat typing indicator (dot animation) is replaced with a static "..." text
- Page transitions use simple fade (not slide)

### Captions for Voice Responses

When TTS is active (the agent speaks its response aloud), a live caption track is always
displayed in the chat bubble — regardless of the voice-first mode setting. This ensures
hearing-impaired users can follow voice responses without needing the audio.

The caption track is the same streaming text shown in the chat view. No extra work is
needed — the text output is always visible alongside the audio.

### Braille Display

Braille display support falls out of TalkBack (Android) and VoiceOver (iOS/macOS)
automatically — Karmik's semantic labels are exposed to the platform accessibility APIs
that drive braille displays. No special handling is required.

### Minimum Touch Target Size

All interactive elements: minimum 48×48dp (Android) / 44×44pt (iOS). Enforced by linting
the widget sizes in the design system. Smaller elements (icon-only buttons) use an invisible
`GestureDetector` wrapper to expand the touch target.

### Accessibility Testing Checklist

Run before each release:
- [ ] TalkBack/VoiceOver: all screens navigable by swipe gestures only
- [ ] Keyboard: all screens navigable by Tab + arrows only (desktop)
- [ ] Font 200%: no text clipped; no layout overflow
- [ ] High contrast: all text meets 4.5:1 ratio
- [ ] Color blind sim: no information conveyed by color alone
- [ ] Reduce motion: no animations play when disabled

---

## Localization (i18n)

### String Externalization

All user-facing strings are externalized to **ARB files** (`Application Resource Bundle`),
Flutter's standard i18n format:

```
lib/
  l10n/
    app_en.arb      ← source strings (English)
    app_es.arb      ← Spanish
    app_fr.arb      ← French
    app_de.arb      ← German
    app_ar.arb      ← Arabic (RTL)
    ...
```

**Generated code**: `flutter gen-l10n` generates a type-safe `AppLocalizations` class.
Usage in code: `AppLocalizations.of(context)!.sendMessage` (never hardcoded strings in
UI widgets).

**Lint rule**: a custom `flutter_lints` rule prohibits string literals directly in
`Text()` widgets (outside the ARB system). Enforced in CI.

### Initial Locales

14 locales at launch:

| Code | Language | Direction |
|---|---|---|
| `en` | English | LTR |
| `es` | Spanish | LTR |
| `fr` | French | LTR |
| `de` | German | LTR |
| `pt` | Portuguese | LTR |
| `ja` | Japanese | LTR |
| `ko` | Korean | LTR |
| `zh-Hans` | Simplified Chinese | LTR |
| `zh-Hant` | Traditional Chinese | LTR |
| `ar` | Arabic | RTL |
| `he` | Hebrew | RTL |
| `hi` | Hindi | LTR |
| `ru` | Russian | LTR |
| `tr` | Turkish | LTR |

### RTL Support

Arabic and Hebrew require right-to-left layout mirroring. Flutter handles most of this
automatically when `Directionality` is set from the locale, but Karmik must:

- Use `start`/`end` (not `left`/`right`) in `EdgeInsets`, `CrossAxisAlignment`, etc.
- Mirror custom icons that are directional (e.g., send arrow → reversed in RTL)
- Flip the overlay bubble anchor from bottom-right to bottom-left in RTL locales
- Test all screens with `Directionality(textDirection: TextDirection.rtl, child: ...)`

A CI test renders each screen with an RTL locale and fails if overflow errors occur.

### Locale Detection

1. Read OS locale (`window.locale` on Flutter Web; `Localizations.localeOf(context)`)
2. If supported: use that locale automatically
3. If not supported: fall back to `en`
4. User can override in Settings → Language → "Display language" (dropdown of all 14 locales)

The language setting is per-device (not synced — users may want different UI languages on
different devices).

### Pluralization and Gender

ARB files support ICU message syntax for plurals and gender:

```json
{
  "messageCount": "{count, plural, =0{No messages} =1{1 message} other{{count} messages}}"
}
```

All count-based strings use ICU plural syntax. Languages with complex plural rules (Russian,
Arabic with 6 plural forms) are handled correctly by the `intl` package.

### Number and Date Formatting

All dates, times, and numbers use `intl` package formatters — never hardcoded format strings:

```dart
// Numbers
NumberFormat.compact(locale: locale).format(tokenCount) // "1.2K" or "1,2K" in German

// Dates
DateFormat.yMMMd(locale).format(date) // "Jun 14, 2026" or "14. Juni 2026"

// Durations
DateFormat.Hm(locale).format(time)  // "09:30" or "9:30 AM"
```

All time is displayed in the **user's local timezone** (not UTC). Karmik uses
`DateTime.now().timeZoneName` for display and stores timestamps in UTC internally.

### Community Translation Workflow

1. Source strings in `app_en.arb` are exported to **Crowdin** (hosted translation platform)
2. Community contributors translate on Crowdin
3. Approved translations are synced back to the repo as ARB files via Crowdin GitHub integration
4. CI verifies that no ARB keys are missing in any locale file (missing = falls back to English)
5. Translators are credited in Settings → About → "Translations" (list of contributor names)

**String freeze**: 2 weeks before each release, no new strings are added to `app_en.arb`,
giving translators time to catch up.

### AI Model Prompts

The agent system prompt and built-in agent descriptions are in English only (the model
reasons in English regardless of UI language). Translated system prompts are a future
consideration once quality can be validated per language.

Agent responses: the model responds in whatever language the user writes in. Karmik does
not force the response language. If the user writes in French, the agent responds in French.
The system prompt includes: "Respond in the same language the user writes in."

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| TalkBack/VoiceOver | ✓ | ✓ | ✓ | ✓ (Narrator) | ✓ (AT-SPI2) | ✓ (ARIA) |
| Keyboard navigation | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Haptic feedback | ✓ | ✓ | ~ (Force Touch) | — | — | — |
| Font scaling | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| High contrast | ✓ | ✓ | ✓ | ✓ | ~ | ✓ |
| Reduce motion | ✓ | ✓ | ✓ | ✓ | ~ | ✓ |
| All 14 locales | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| RTL layout | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**Linux high contrast (~)**: detected via `org.gnome.desktop.interface.high-contrast` gsettings;
not available on non-GNOME desktops (KDE, etc.) — falls back to user's manual theme selection.

**macOS haptics (~)**: available on MacBooks with a Force Touch trackpad via
`NSHapticFeedbackManager`; not available on Mac Pro/mini (no haptic actuator).
