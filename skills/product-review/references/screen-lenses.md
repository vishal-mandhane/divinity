# Screen lenses — what to look at, in order

Run these on every screen. Each lens produces cited observations, not scores.
Scoring happens later in the main context (see `scoring-rubric.md`).

Order matters: a screen that fails Lens 1 cannot be saved by Lens 6.

---

## Lens 1 — First glance (3 seconds)

Read the build method top to bottom and answer:

- What is the **one** thing this screen wants the user to do? Cite the widget.
- How many interactive elements sit above the fold? Count buttons, chips,
  tappable cards, `AppBar` actions, tabs, FABs, bottom nav.
- Does the title match the content?
- Is there a second thing competing for the same attention? Cite both.

**Finding shape:** `N interactive elements above the fold; primary action is
<widget> at file:line, competing with <widget> at file:line.`

Above ~7 interactive elements above the fold on a phone, say so plainly. It is a
count, not an opinion.

## Lens 2 — Thumb and placement

The user holds a phone in one hand. Where things sit is a product decision.

- Is the primary action reachable by the thumb (bottom third), or parked in the
  top-right corner?
- Is a destructive action (leave, delete, cancel, report, block) adjacent to a
  common action? Cite both and the gap between them.
- Are tap targets at least 48×48 dp (Android) / 44×44 pt (iOS)? Cite any that
  are not.
- Does anything important sit behind a scroll, a tab, or a long-press with no
  affordance?
- Do bottom sheets and dialogs have a visible way out?

## Lens 3 — The four states

For each of loading / empty / error / success, find the code that renders it or
prove it is absent:

- **Loading** — skeleton, shimmer, or a bare spinner? Is the screen blocked
  while it loads?
- **Empty** — is the copy specific to this screen and does it offer one action?
  "No data" is a defect.
- **Error** — does the user see anything? A `catch` that only writes to a log
  shows the user nothing. Cite the `catch` block.
- **Success** — does the user get confirmation, and does the verb match the
  button they pressed ("Join" → "Joined")?

## Lens 4 — Copy

- Read every user-facing string on the screen. Quote the bad ones with line
  numbers.
- Dev voice ("Submit", "Configure", "Invalid input") → rewrite in one line.
- Does any string state a number, a time window, or a rule? Verify it against
  the code that enforces it. Mismatch = HIGH finding.
- Brand: does every user-visible string use the product's public name and
  spelling? Internal code names leaking into the UI are a bug.
- Would the target user read this sentence out loud without cringing?

## Lens 5 — Data honesty

- What does the screen read, and what does it write? Cite the service call.
- Does it show stale cached data without saying it is stale?
- Does an optimistic update roll back on failure? Cite the rollback or its
  absence.
- Any number shown to the user that is computed client-side and could disagree
  with the server (member counts, scores, unread badges)? Cite both
  sources.

## Lens 6 — Design system fit

If the project has its own design-system doc or UI skill, run its checklist.
Otherwise run the `ui-ux-review` skill's checks. Report violations with line
numbers and stop there. Do not add design opinions this skill does not own.

Minimum checks: colours and fonts hardcoded instead of taken from the theme,
decorative gradients, stacked or coloured shadows, a missing dark-mode branch,
magic-number spacing, a bare spinner as the primary loader, and remote images
with no error fallback.

## Lens 7 — Robustness pressure test

State whether each was checked or not — never assume:

- Long strings (a 40-character title, a long username)
- Large text scale (2.0×) and a 320px-wide phone
- No network / slow network
- Empty account (no content, no history, no connections)
- A user in a region where the app has no content yet

Anything not checked goes in the ledger as unchecked. Do not silently pass it.

---

## Per-screen output block (agents must return exactly this)

```
### <file path>
LINES: <total line count>  READ: <full | lines a-b of n>
PURPOSE: <one line, from the code>
REACHED FROM: <file:line of each Navigator call that opens it, or NOTHING>
ABOVE-FOLD ELEMENTS: <count>
PRIMARY ACTION: <widget @ file:line>
STATES: loading=<yes/no @line> empty=<...> error=<...> success=<...>
COPY PROBLEMS: <"quote" @line → suggested rewrite>  (or NONE)
UI VIOLATIONS: <design rule @line>  (or NONE)
TRUST: <consequence copy + appeal path @line, or MISSING>
UNCHECKED: <pressure tests not run>
OBSERVATIONS: <3+ bullets, each ending in file:line>
```

No scores. No adjectives without a line number. If a field cannot be filled
from the code, write `UNKNOWN` — never guess.
