# Report templates

Write to `docs/product-review/YYYY-MM-DD/`. Seven files. Use these skeletons.

`00-REVIEW.md` is written for the owner, who does not write code. Short
sentences. Name files and lines. Say "this is broken" or "this is fine". Give
the recommendation, not a menu.

---

## 00-REVIEW.md

```markdown
# <App> product review — <date>

## Verdict
<Two sentences. What the app is good at today, what is holding it back.>

**Product score: X.X / 5** (weighted: core loop ×3, onboarding/auth ×2,
admin/settings ×1)

| Area | Score | Band |
|------|-------|------|
| <Core action> | X.X | Fix |
| <Creation flow> | X.X | Rework |
| <Retention loop> | X.X | Ship |
| Onboarding & auth | X.X | Rework |
| Trust & safety | X.X | Fix |
| Admin | X.X | Ship |

## The 10 things to do next

| # | Do this | Where | Effort | Impact | Why |
|---|---------|-------|--------|--------|-----|
| 1 | <one action> | `file:line` | S/M/L | High/Med/Low | <one line> |

Effort: S = under an hour, M = a day, L = more than a day.
Ranked by impact ÷ effort. Number 1 is what to do today.

## What is already good
<3–6 bullets, each with a file:line. Do not skip this section — it stops the
review from reading as a hit piece and it marks what not to touch.>

## Questions only you can answer
<The levers where the trade is a business call. One line each, phrased as a
choice with your recommendation attached.>

1. **<Lever>** — keep <X>, or change to <Y> because <trade>?
   *Recommendation: <keep/change>, and <structural fix first, e.g. extract the
   constant (N hardcoded sites)>.*

## What we did not check
<Straight list. Screens not read, flows not traced, pressure tests not run.
See 05-EVIDENCE.md for the ledger.>
```

---

## 01-FEATURES.md

```markdown
# Feature inventory — <date>

Screens: N (counted: `<the enumeration command>`)
Features: M | Backend entry points: K

## Inventory

### <Feature name>
**Verdict: KEEP / FIX / MERGE / CUT** — Reachable X, Complete X,
Understandable X, Worth-the-code X (mean X.X)

- **What it does (PROVEN):** <one line> — `file:line`
- **Screens:** `path`, `path`
- **Service / backend:** `file:line`, `<backend file>:line`
- **Writes:** `collection.field` — `file:line`
- **Reached from:** `file:line`, or **NOTHING NAVIGATES HERE**
- **Unfinished parts:** `TODO @ file:line`, stubbed `file:line`, or none
- **Judgement:** <why the verdict>
- **If CUT, the tax it keeps alive:** <files, lines, functions, collections,
  indexes>

## Orphans
<Screens and services nothing routes to. Each with the grep that proves it.>

## Overlaps
<Features whose jobs collide. Cite both. Recommend which one wins.>
```

---

## 02-FLOWS.md

```markdown
# Flows — <date>

## <Flow name>
**Verdict: TOO LONG / RIGHT / TOO THIN**
Screens: N · Required inputs: M · First value at step: S of N

| Step | Screen | Gets here from | User must give | Blocking wait |
|------|--------|----------------|----------------|---------------|
| 1 | `<login screen>` | app start `file:line` | nothing | none |

**Branches:** <condition @ file:line → destination>
**Dead ends:** <state @ file:line, what the user sees, no way forward>

**What is wrong:** <plain English, counts not vibes>
**Fix:** <merge / defer / delete which specific steps, in order>

> Defer beats delete. Moving a question to after first value keeps the data and
> removes the leak.
```

---

## 03-SCREENS.md

```markdown
# Screen scorecard — <date>

Sorted worst first. Covered N of TOTAL — gaps listed in 05-EVIDENCE.md.

| Screen | Clarity | Effort | Feedback | Fit | Trust | Payoff | Score | Band |
|--------|:-------:|:------:|:--------:|:---:|:-----:|:------:|:-----:|------|
| `path/to/screen` | 2 | 2 | 3 | 4 | 1 | 3 | 2.5→2.9 cap | Rework |

Any axis at 1 caps the screen at 2.9. Mark capped rows.

## Details (every screen below 4.0)

### `path/to/screen` — X.X, <Band>
- **Job:** <one line> — `file:line`
- **Reached from:** `file:line`
- **Evidence:** <3+ cited observations>
- **What to do:** <numbered, ordered, each naming the file:line to touch>
- **Unchecked:** <pressure tests not run>

## Screens at 4.0+
<One line each: name, score, the polish win.>
```

---

## 04-LEVERS.md

```markdown
# Product decision register — <date>

Every tunable value someone chose. Owner decides; this file recommends.

## <Lever name> — current value: <X>

| | |
|--|--|
| **Set at** | `file:line` ×N — **N sites = defect** |
| **Promised to users at** | `file:line`, or none |
| **Owner** | Founder / Trust & Safety / Growth / Ops |
| **Optimizes for** | <one line> |
| **Too low breaks** | <what fails> |
| **Too high breaks** | <what fails> |
| **PROVEN recommendation** | <structural: extract constant, align copy, add a check> |
| **JUDGEMENT recommendation** | <keep / change to X, and why> |
| **Settles it** | <the one metric or experiment> |

## Levers that are correct — leave them alone
<Name, value, one line on why it is right. This section prevents re-litigation.>

## No data
<Levers where no measurement exists. Recommend the instrumentation, not a value
change.>
```

---

## 05-EVIDENCE.md

```markdown
# Evidence & coverage — <date>

## Commands run
```
<command>
<raw output or the count it produced>
```

## Coverage ledger
| Surface | Total | Covered | Read in full | Not checked |
|---------|------:|--------:|-------------:|-------------|
| Screens | 58 | 58 | 51 | <list the 7> |
| Services | 55 | 30 | 12 | <list> |
| Backend entry points | 79 | 79 | 20 | <list> |
| Flows | 6 | 6 | — | — |

## Verification
Rows submitted: N · CONFIRMED: N (X%) · WRONG-LINE: N (fixed) ·
NOT-FOUND: N (deleted) · CLEARED: N

Below 90% confirmed means the pass was sloppy. Say so if it happened.

## Deleted claims
<Every row the verifier killed, and why. Publishing these is the point — it
proves the rest was checked.>

## Assumptions
| Assumption | Why we needed it | The metric that would settle it |
|------------|------------------|--------------------------------|

## Agents used
| Role | Count | Shards | Returned complete |
|------|------:|--------|-------------------|

## Not checked, and why
<Plain list. Truncation, missing access, flows needing a device.>
```

---

## 06-SERVICES.md

```markdown
# Service layer — the rules behind the screens

## The headline
<The one gate that behaves differently from what the screens imply. If there
isn't one, say so — but look hard first.>

## The <core action> pipeline, in order
| # | Check | Line | What the user sees |
|---|-------|------|--------------------|
| 1 | <condition> | `file:line` | <quote, or **generic failure**> |

<Count how many refusals have their own copy versus how many collapse into one
undifferentiated failure. A tap that silently does nothing is the most
confusing thing an app can do.>

## Silent side effects — the full list
| What happens | Where | Told? |
|--------------|-------|-------|
| <change to the user's record> | `file:line` | no |

<This table is the evidence base for any "the app punishes people silently"
claim in 00-REVIEW. Do not make that claim without it.>

## Rules enforced in only one place
<Client-only rules (bypassable) and rules-only rules (the client comment lies).
Both matter, for opposite reasons. Hand client-only security rules to
`security-audit`.>

## Dead code and duplicate implementations
<Services nothing calls. Two code paths doing one job with different
side effects — name what the shortcut skips.>

## Error handling
<Which services swallow errors into false/[]/null, and which surface them.
If screens show "nothing here" on failure, the root cause is usually here.>

## New levers found here
| Lever | Value | Where | Note |
|-------|-------|-------|------|

## What this changes elsewhere in the report
<Numbered list of edits made to the other six files as a result. If a screen
score moved, say which and why. If none moved, say that.>
```

---

## Style rules for all seven files

1. Plain English. Short sentences. One idea each.
2. Every fact ends in `file:line`.
3. PROVEN and JUDGEMENT labelled separately wherever both appear.
4. No banned words: might, could, consider, potentially, seems, appears, robust,
   seamless, leverage, comprehensive.
5. No executive filler. If a section is one line, it is one line.
6. If the run was cut short, the last line of `00-REVIEW.md` is exactly:
   `TRUNCATED AT [area] — [N] screens / [M] features unchecked`
