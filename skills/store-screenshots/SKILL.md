---
name: store-screenshots
description: Use when producing Google Play or App Store listing screenshots, a store carousel, a feature graphic, or social 4:5 slides cut from the same source — including choosing which screens to show, writing the caption on each, and exporting at store-legal sizes.
---

# Store Screenshots

## Overview

The store carousel is the highest-leverage marketing surface an app has, and the
part that gets shipped wrong is almost never the taste. Baseline testing on a
doc-rich repo showed the creative work — narrative order, positioning, caption
voice — comes out good unaided. What came out broken every time: **dimensions
that violate store rules, on-screen strings quoted from memory instead of from
source, and features framed that are not in the shipping build.**

This skill is about those three. Do not let it flatten the copy into a template;
the copy should be specific to this product or it is worthless.

## When to Use

- Building the Play Store or App Store listing screenshot set
- Writing the caption/headline that sits on each screenshot
- Producing the 1024×500 Play feature graphic
- Re-cutting the same set as 4:5 social slides

**Not for:** launch videos, in-app onboarding, or press kits.

## Rule 1 — Frames 1–3 carry the entire pitch

Published ASO benchmarks all point the same way, even though their exact
numbers disagree. Treat the numbers as direction, not fact — they come from
vendors (StoreMaven, ASO tool blogs), not from Apple or Google:

| Commonly cited | Consequence |
|---|---|
| Most visitors (figures range 70–83%) never scroll past screenshot 1 | Frame 1 alone must answer *what is this* |
| Very few (figures around 10%) see past screenshot 3 | The pitch is complete by frame 3 |
| About 2.5 frames sit above the fold on a large phone | Frame 3 is half-visible; it is a hook, not a payoff |
| Only a few percent swipe the whole gallery | Frames 4+ reach a minority already leaning in |

If you quote any of these to a client or in a deck, cite the vendor study you
took it from.

So: **frame 1 = what it is and who it is for. Frame 2 = the differentiator.
Frame 3 = the objection that arrives once someone believes frames 1–2.** Frames
4–8 are depth, utility and breadth, in that order.

The common failure is writing the set as an even feature tour across 8 frames.
That spends the two frames that matter on a table of contents.

**One benefit per frame.** A frame that needs "and" in its headline is two frames.

**Caption word cap: 6 words for the headline, one line for the subline.** If it
does not survive Rule 6, cut words — never shrink the type.

## Rule 2 — No quote without a citation

Every string that appears on a panel as product copy — a headline pulled from the
UI, a feature name, a number, a label — **must be grepped from source first**, and
you record `file:line` next to it in the plan.

This is not pedantry. In baseline testing an agent put the sentence
`"it is 3:40 am for rhea"` on a panel and cited a line number. The real string was
`'it is the middle of the night for '` at a different line. Invented UI copy on a
store asset is a misrepresentation that ships.

Writing *new* marketing copy is fine and expected. Presenting invented text **as
the app's own words** is the violation.

## Rule 3 — Every framed feature must exist in the shipping build

For each frame, name the artifact that proves the feature ships: the widget
provider, the Cloud Function, the release-flavour entry. A feature that works in
debug, or whose backend needs an account you do not have yet, **cannot be framed**.

Check specifically for features gated on: a paid backend tier, a developer account
you have not bought, a store-billing integration, or an API key. These are the ones
that look done in the repo and are not live for a user.

If a feature is not verifiable, move it to the bench and use a replacement frame.

## Rule 4 — Comp per store; never resize between them

**Apple's tall portrait is not a legal Play size.** Apple 6.9" is 1320×2868
(≈2.17:1). Play requires the **longest side ≤ 2× the shortest**. A 2.17:1 image
fails that rule, so an Apple master cannot be resized into a Play asset — the
caption block has to be re-composed for 9:16.

Full tables: `references/store-specs.md`. The traps, inline:

- Play: min side 320px; max side 3840px; **longest ≤ 2× shortest**
- Play: for high-visibility placements, use 9:16 portrait at **≥ 1080×1920**
  (or 16:9 landscape ≥ 1920×1080) — so in practice, build at 9:16
- Play: JPEG or **24-bit PNG with no alpha channel**; ≤8MB each; up to 8 per device type, at least 2 in total to publish
- Play: the **1024×500 feature graphic is required**, and its centre must stay clear
  because a play button overlays it once a promo video exists
- Apple: 6.9" master at 1320×2868 auto-scales to smaller iPhones; 1–10 per class
- Apple: 1290×2796 is the older 6.7" size, accepted but not the current master

## Rule 5 — Nothing in the art that policy reads as a claim

Banned from the pixels, regardless of whether it is true:

- Price, "free", "free forever", trial lengths, discounts
- CTAs: "download now", "get it", "install"
- Rankings, superlatives, testimonials: "#1", "best", "as seen in"
- Security words you cannot back: "private", "secure", "encrypted" unless it is
  genuinely end-to-end and you can prove it
- Any content that raises the age rating: adult pack names, real-money stakes,
  gambling-shaped mechanics. Listing art is seen by everyone regardless of rating.

Put the free-tier promise in the long description, where it belongs, not baked
into a PNG you cannot edit without a re-upload.

## Rule 6 — Proof at thumbnail before export

Lay the full set out at **150px wide** in a row and read it. If a headline is
illegible there, cut words. Also check the strip has a **colour beat** — eight
frames on the same ground read as one grey block in search results.

Check the frame edge survives against both a **white and a dark** store card. A
light background with no boundary dissolves into Play's white listing card.

## Rule 7 — One state, one clock, every frame

Render every frame from the same seeded/demo app state with a pinned clock. Same
partner name, same counts, same balances, same dates. Numbers that drift between
frames read as mockups and destroy the credibility the real UI bought you.

## Red Flags — stop and fix

- A headline containing "and" → two frames
- A quoted UI string with no `file:line` beside it → grep it or cut it
- "We'll resize the Apple set for Play" → illegal aspect ratio
- A frame for a feature whose backend isn't live → bench it
- 8 frames of equal weight → frames 1–3 aren't doing their job
- Currency or names that only suit one market → build a locale variant
- Alpha channel in the export → Play rejects it

## Rationalizations

| Excuse | Reality |
|---|---|
| "The dimensions are close enough" | Play enforces the 2× rule at upload. Close enough is rejected. |
| "I remember that string from the code" | You don't. Grep it. Baseline testing caught exactly this. |
| "The feature's basically done" | If it needs a card on file or an account you lack, it is not in the build. |
| "Eight frames means eight features" | 90% never see frame 4. Spend the budget where the eyes are. |
| "Legal wording is the lawyer's problem" | Store review rejects the asset; you re-upload and lose days. |
| "It looks fine on my monitor" | Proof at 150px. That is the size that decides installs. |

## Process

1. **Read the product's own positioning docs first** — plan, roadmap, design-system
   comments. A repo that argues its own positioning will out-write anything generic.
2. Inventory available screen renders and note each one's true pixel size; do not
   upscale a 1x render.
3. Draft the frame order against Rule 1, then write captions with citations (Rule 2).
4. Run the gates: Rule 3 (ships?), Rule 5 (policy?), locale variants.
5. Render source states at master resolution — `references/render-pipeline.md`.
6. Comp, proof at thumbnail (Rule 6), export per store (Rule 4).

## Why it matters

Screenshots are usually the cheapest conversion work available, and unlike the
icon they can be tested: Google Play has store listing experiments, and the App
Store has product page optimization. Run one before arguing about taste.
