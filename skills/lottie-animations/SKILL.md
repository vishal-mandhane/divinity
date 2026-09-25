---
name: lottie-animations
description: Use when asked to suggest Lottie animations for a Flutter screen, or when reviewing a screen file path for places where motion design could replace static loaders, generic spinners, empty states, success confirmations, or transitions. Use when the user wants vivid, non-cliche micro-animation ideas instead of stock CircularProgressIndicator-style placeholders.
---

# Lottie Animations

## Overview

**Motto:** "A spinner is a placeholder. A character doing a somersault is a story."

Given a Flutter screen file, find every moment where motion could ship — loaders, RefreshIndicators, empty states, success/error confirmations, transitions, onboarding illustrations, tap-feedback hero moments — and propose **specific, character-driven or geometric-narrative** animation ideas with the Flutter code to wire them in.

The whole point of this skill is to **never suggest a generic animation**. If the suggestion could appear unchanged in any other app, it has failed.

## When to Use

- User points at a screen file (e.g. `lib/screens/<file>.dart`) and asks "what animations could go here"
- Reviewing a screen with `CircularProgressIndicator`, `RefreshIndicator`, empty placeholder Text, or Snackbar/AlertDialog success states
- Onboarding screens with static illustrations
- Any moment the user describes as "feels flat" or "needs life"

## Method

1. **Read the screen file in full.** Never analyse from a snippet. Note the screen's purpose (login? feed? checkout? empty inbox?) and the user's emotional state at each moment (anticipation, frustration, delight, confusion).
2. **Check for a project adapter file** at `<repo>/.claude/lottie-context.md`. If present, load brand colors, the project's lottie wrapper widget, asset path conventions, and domain vocabulary. Use these in every snippet and concept.
3. **Scan for animation moments** (see checklist below). For each, capture `file:line`, the trigger, and the user's emotion at that beat.
4. **For each moment, generate 2–3 concepts** across different aesthetics (playful character / minimalist geometric / brand-tied) using the lenses below. Reject the first idea that pops up — it's almost always a cliche.
5. **Pick a delivery format per moment:** Lottie, Rive (interactive), or pure-Flutter (CustomPainter / AnimatedBuilder). Recommend the cheapest tool that ships the feeling.
6. **Write the integration snippet** wired to the right widget (empty-state Column, Snackbar content, loader slot) at the exact `file:line`. Note: Flutter's built-in `RefreshIndicator` only draws its own spinner — it has no builder for custom art. For custom art, use `RefreshIndicator.noSpinner` (in recent Flutter; it hides the spinner and reports drag state through `onStatusChange`) and draw the Lottie yourself, or use a package such as `custom_refresh_indicator`. Say which one the snippet uses and check the project's Flutter version.

## Animation-Moment Checklist

Scan the file for these widget patterns:

| Widget / Pattern | Moment Type | User emotion |
|---|---|---|
| `CircularProgressIndicator`, `LinearProgressIndicator` | Loading | Waiting, anticipation |
| `RefreshIndicator` | Pull-to-refresh | Active demand for newness |
| Empty-state `Text` ("No events yet", "No friends") inside `if (list.isEmpty)` | Empty state | Mild disappointment, opportunity |
| `Snackbar`, `AlertDialog`, `showDialog` on success | Confirmation reward | Delight, completion |
| `try/catch` UI with error `Text` / red banner | Error | Frustration, confusion |
| `Hero` widgets and route transitions | Transition | Continuity |
| Onboarding `PageView` static illustrations | Illustration | Curiosity |
| Long-press, tap-and-hold, drag dismiss | Tap-feedback hero | Tactile satisfaction |
| Form submit / button press → API call | Action acknowledgement | "Did it work?" |
| Background polling / silent updates | Ambient liveness | Reassurance app is alive |

## Concept-Generation Lenses

Run each moment through these — the second and third lens always beat the first.

1. **Replace the verb literally.** "Refresh" isn't *spinning* — it's renewal, upward motion. Use somersault, rocket-launch, paper-airplane-relaunch, gymnast-vault, balloon-release.
2. **Tie to the domain.** What does the app DO? An events app reaches for crowds, party hats, dance moves, fireworks, ticket stubs. A finance app reaches for coins, jars, ledger lines. A fitness app reaches for heartbeats, sweat drops, lungs. **Domain-tied animations feel custom even when they're not.**
3. **Subvert the cliche.** Loaders are circles → use a juggler, a potter, a knitter, a barista pulling espresso. Success is a checkmark → use a tiny crowd raising a flag, a confetti pop tied to brand colour, a mascot's fist-pump. Empty states are sad clouds → use *anticipation*: empty stage with a spotlight, a doorman waiting, a swept dance floor.
4. **Physics over linear tweens.** Squash-and-stretch, elastic bounce, gravity drop, momentum carry. Linear easing reads as "templated".
5. **Single-take micro-narrative.** Every animation is a 1.5–3 second story: setup → action → exit. Not just a looping motion blob.
6. **Geometric route for "premium" surfaces.** When playful would feel wrong (checkout, settings, formal moments), bias toward morphing primitives, line-art that draws itself, abstract masses easing into place. Still anti-generic: no abstract "swirls".

## Anti-Cliche Banned List

Never suggest these without an explicit reason:

- Spinning circles / abstract loaders / generic shimmer for loading
- Plain checkmark drawing-in for success
- Sad clouds, rainclouds, or single tear-drops for empty/error
- Generic confetti rain (specify the *shape*, *colour palette*, and *origin point*)
- Hourglass, ticking clock for waiting
- Spinning gear for "processing"
- Heart pop for "liked" (unless explicitly themed to the brand mascot)

If the user explicitly wants minimal, premium, or boring — say so in the recommendation but propose at least one specific non-generic version anyway.

## Tool Choice (Lottie vs Rive vs pure-Flutter)

- **Lottie** — pre-composed, designer-authored, one-shot or loop, no user input. Cheap to ship. Default choice.
- **Rive** — user input drives the animation (drag a slider and see the character react). Use only when interactivity is the point.
- **Pure-Flutter (CustomPainter / AnimatedBuilder / AnimatedContainer)** — geometric, parameter-driven, brand-colour-aware. Better than Lottie when: the motion is just a property tween (height, opacity, colour), the colours must adapt to theme dynamically, or the asset would be < 5KB of code.

Always mention the pure-Flutter alternative when it would actually be cheaper.

## LottieFiles Search Keywords

I cannot verify specific LottieFiles URLs exist or guarantee a given animation is on the site. **Output search keywords, not direct asset links.** Format: 3–5 short search phrases the user can paste into lottiefiles.com.

Example for a somersaulting-monkey refresh:
- `monkey somersault`
- `acrobat flip loop`
- `gymnast vault refresh`
- `character backflip transparent`

## Output Format

For each animation moment found in the screen:

```
### Moment N: <descriptive name> — `file.dart:LINE`

**Trigger:** <exact event that fires it>
**User emotion:** <one-line>
**Current state:** <what's there now, e.g. "CircularProgressIndicator centred">

**Concept A — Playful character:**
<2-3 sentences. Specific creature, specific physical action, specific physics, specific colour cue.>

**Concept B — Minimalist geometric:**
<2-3 sentences. Specific shapes, specific morph, specific easing.>

**Concept C — Brand-tied / domain:**
<2-3 sentences. Pulls from the app's actual domain vocabulary.>

**Recommended:** <A | B | C> — <one-line why>

**LottieFiles search keywords:** "...", "...", "..."

**Flutter snippet:**
\`\`\`dart
// <file.dart:LINE — replaces <what>>
<exact replacement code, wired to the right widget>
\`\`\`

**Non-Lottie alternative:** <only if cheaper/better — say which Flutter widget and why; otherwise omit this line>
```

Close with a **Summary** section listing the recommended concept per moment in one line each, so the user can scan and pick.

## Common Mistakes

- **Reading only a snippet of the file.** You miss half the moments. Read the whole screen.
- **Suggesting "a Lottie animation" without describing what's in it.** "Add a Lottie here" is not a suggestion. Describe the character, the action, the colour, the physics, the duration.
- **One aesthetic per moment.** Always give the user 2–3 options across aesthetics so they can pick what matches the surface.
- **Asserting a specific LottieFiles URL exists.** You cannot verify this. Use search keywords, not URLs.
- **Ignoring dark mode.** If the project is dark-mode-first, every concept must specify how it reads on dark backgrounds (stroke weight, colour shift, glow).
- **Generic success animations after a specific action.** "Event joined" deserves a crowd-cheer, not a green tick. Tie the reward to the verb.
- **Suggesting Lottie for what is obviously a 6-line `AnimatedContainer`.** Reach for the cheapest tool.

## Quick Reference: Cliche → Specific

| Generic | Specific replacement direction |
|---|---|
| Spinner loader | Juggler, potter at wheel, barista pull, knitter, painter brush-stroke |
| Pull-to-refresh swirl | Somersault, rocket launch, balloon release, paper plane loop |
| Checkmark success | Action-tied micro-reward (crowd flag-raise, confetti from button origin, mascot fist-pump) |
| Sad cloud empty state | Anticipation (empty stage + spotlight, doorman waiting, swept floor, single chair) |
| Generic error face | Polite stumble (paper plane crash-land, mascot scratching head, dropped tray) |
| Confetti rain | Brand-coloured shapes from the action's origin point, specific count, specific arc |
| Hourglass waiting | Domain-tied wait (queue line shuffling, vinyl spinning, kettle steam) |

## Remember

- **Specificity is the entire product.** "A character does X with Y physics in Z colour" beats "a fun loader".
- Always offer the **pure-Flutter alternative** when the motion is just a property tween.
- Always ground the concept in **the user's emotion at that exact beat**, not the widget type.
