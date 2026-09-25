---
name: ui-ux-review
description: Use when reviewing or changing any app UI for accessibility, responsiveness, dark mode, motion or loading states — screen reader labels, touch target sizes, contrast ratios, text scaling, overflow on small screens, reduced motion, and missing empty/error states. Invoke after building or editing a screen or component, before shipping a UI change, or when asked "is this accessible", "does this work in dark mode", "check for overflow", or "review this screen". Examples are Flutter; the checks apply to any stack.
---

# UI/UX Review

## Overview

**If a user can't see it, reach it, or understand it, it isn't there.**

This skill checks one screen or component against **measurable** rules. Every
finding names a number from a published standard, the `file:line` that breaks
it, and who is hurt by it. "Looks a bit cramped" is not a finding.

**Read the whole file first.** A screen's error state is usually 300 lines below
its happy path.

## The Iron Laws

1. **Cite the standard, not taste.** Each rule below has its source. If a
   finding has no number and no source, it goes under "Polish", not "Issues".
2. **Every finding has `file:line`** from a file you opened this session.
3. **Checked or unchecked, never assumed.** If you did not render it at 2× text
   or on a 320-wide screen, say "not rendered" — do not pass it.
4. **The project's design system wins on style.** If the repo has a theme file,
   design tokens, or a design-system doc, hardcoded values that bypass it are
   findings. This skill never overrides the project's own colours or fonts.

## 1. Accessibility (the numbers)

| Check | Rule | Source |
|---|---|---|
| Text contrast | **4.5:1** for normal text, **3:1** for large text (≥ 18pt, or ≥ 14pt bold) | WCAG 2.2 SC 1.4.3 (AA) |
| Non-text contrast | **3:1** for icons, input borders, focus rings, and state indicators against their background | WCAG 2.2 SC 1.4.11 (AA) |
| Touch target | **48×48 dp** Android, **44×44 pt** iOS. WCAG's floor is 24×24 CSS px (SC 2.5.8), but platform guidance is the real bar on phones | Material Design; Apple HIG |
| Text resize | Content still works at **200%** text size — no clipping, no overlap, no lost actions | WCAG 2.2 SC 1.4.4 |
| Flashing | Nothing flashes more than **3 times per second** | WCAG 2.2 SC 2.3.1 |
| Colour alone | State (error, selected, required) is never shown by colour only — add an icon, text, or shape | WCAG 2.2 SC 1.4.1 |
| Names | Every interactive element has an accessible name a screen reader speaks | WCAG 2.2 SC 4.1.2 |

Contrast is computed, not eyeballed: take the two hex values and run the WCAG
relative-luminance formula, or a contrast checker. Report the actual ratio
("#8A8A8A on #FFFFFF = 3.5:1, needs 4.5:1").

**Screen readers — what to look for (Flutter terms, same idea everywhere):**

- `IconButton` with no `tooltip` → announced as just "button". Add `tooltip`.
- `GestureDetector` / `InkWell` used as a button → no button role. Wrap in
  `Semantics(button: true, label: ...)` or use a real button widget.
- `Image` with meaning but no `semanticLabel`; decorative images not excluded
  (`excludeFromSemantics: true`).
- Custom toggles/checkboxes with no `toggled` / `checked` semantics.
- Reading order that jumps around because of `Stack` / absolute positioning.

Web: the same checks as `aria-label`, `role="button"`, `alt`. React Native:
`accessibilityLabel`, `accessibilityRole`. SwiftUI: `.accessibilityLabel`.

## 2. Text scaling and small screens

Test these two conditions, or mark them "not rendered":

- **Text at 2.0×.** In Flutter:
  `MediaQuery(data: MediaQuery.of(context).copyWith(textScaler: TextScaler.linear(2.0)), child: ...)`.
- **320 dp wide.** The smallest common phone width.

What breaks, in order of how often:

- Fixed-height containers around text → text clipped. Let height grow, or use
  `minHeight`.
- `Row` with two `Text`s and no `Expanded`/`Flexible` → overflow stripes.
- Hardcoded widths (`width: 400`) → off-screen on small phones.
- Buttons whose label wraps and pushes the icon out.
- `maxLines: 1` on anything a user types (names, titles) with no
  `overflow: TextOverflow.ellipsis`.

Clamp text scaling only as a last resort and never below the platform's
accessibility sizes — clamping is how apps quietly fail low-vision users.

Responsive layout: branch on the **available width**, not the device type.

```dart
LayoutBuilder(
  builder: (context, constraints) {
    // Material window-size classes: compact < 600, medium < 840, expanded ≥ 840
    if (constraints.maxWidth >= 600) {
      return Row(children: [sidebar, Expanded(child: content)]);
    }
    return content;
  },
)
```

## 3. Dark mode

- Every colour comes from the theme (`Theme.of(context).colorScheme`, or the
  project's tokens). A literal `Colors.white` / `Color(0xFF...)` in a widget is
  a finding — it will be wrong in one of the two modes.
- Check **both** modes for contrast. A colour that passes 4.5:1 on white often
  fails on near-black.
- Avoid pure black `#000000` surfaces behind pure white text for long reading —
  the halo is tiring. Material's dark baseline surface is `#121212`.
- Shadows barely show on dark surfaces. Elevation in dark mode is expressed with
  lighter surface tones, not bigger shadows.
- Images and illustrations with white backgrounds glow in dark mode. Check them.

## 4. Motion

- **Respect reduced motion.** In Flutter,
  `MediaQuery.disableAnimationsOf(context)` is true when the user turned
  animations off. Non-essential motion (parallax, auto-playing loops, big
  transitions) must stop or become a fade.
- **Durations by size:** small UI feedback (a toggle, a ripple) 100–200 ms;
  element transitions 200–300 ms; full-screen transitions up to ~500 ms.
  Anything longer makes the user wait for decoration.
- Performance: animate with `AnimatedBuilder` / implicit animations so only the
  animated subtree rebuilds; put a `RepaintBoundary` around a continuously
  animating widget that sits inside a static screen.
- No flashing over 3 per second (see §1).

## 5. The four states

Every screen that loads data needs all four. Find the code for each, or prove it
is missing.

| State | Pass | Fail |
|---|---|---|
| Loading | Skeleton or placeholder shaped like the content; the rest of the screen still usable where possible | A lone spinner in the middle of a blank screen for more than a moment |
| Empty | Says **why** it is empty and gives **one** next action | "No data" / blank space |
| Error | Plain-language message + a retry; the error is shown, not only logged | A `catch` that only logs, leaving the user on a spinner or a blank list |
| Success | Confirms the action with the same verb as the button ("Save" → "Saved") | Nothing changes, so the user taps again |

```dart
if (isLoading) return const ContentSkeleton();
if (error != null) return ErrorState(message: error!, onRetry: load);
if (items.isEmpty) return EmptyState(onCreate: openCreate);
return ContentList(items: items);
```

## 6. Pressure tests

State each as checked or not checked:

- A 40-character title and a 30-character username
- Text at 2.0× and a 320-wide screen
- Dark mode and light mode
- No network, and a slow network (does loading ever end?)
- An empty account (no content, no history)
- Screen reader on: can you complete the screen's main action by ear alone?

## Output Format

```markdown
# UI/UX Review — <screen> — <date>

Rendered: <which pressure tests you actually ran> | Not rendered: <the rest>

## ♿ Accessibility
### A-1 <title>
**Where:** `path/to/file:LINE`
**Rule:** <standard + number, e.g. WCAG 1.4.3, 4.5:1>
**Measured:** <e.g. 3.1:1, 36×36 dp, no label>
**Who is hurt:** <e.g. low-vision users, screen-reader users>
**Fix:** <exact change>

## 📱 Text scaling & small screens
## 🌙 Dark mode
## 🎬 Motion
## ⏳ States (loading / empty / error / success)
<same shape>

## ✨ Polish (no standard broken)
<taste-level suggestions, clearly separated>

## ✅ Checked and fine
<what you checked that passed — name the check>
```

## Red Flags — stop and fix the review

- A finding with no number or no source → move it to Polish
- "Contrast looks low" with no ratio → compute it
- Passing text scaling without rendering it
- A colour fix that hardcodes a new hex instead of using the theme
- Reviewing only the happy path of a screen that has a `catch` block

## Quick Reference

| Check | Number |
|---|---|
| Normal text contrast | 4.5:1 |
| Large text / icons / borders | 3:1 |
| Touch target | 48×48 dp (Android), 44×44 pt (iOS) |
| Text scale to test | 200% |
| Narrowest width to test | 320 dp |
| Flashing limit | 3 per second |
| Small feedback motion | 100–200 ms |
| Screen transition | ≤ ~500 ms |
