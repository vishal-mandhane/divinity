---
name: product-review
description: Use when reviewing an app as a product rather than as code — "is this feature good enough", "audit all the screens", "rate the screens", "is the signup flow too long", "should this limit be 5 or 3", "what should we cut", "product audit", "PM review", or a pre-launch feature review. Produces an evidence-cited feature inventory, flow maps, a 1–5 scorecard for every screen and feature, and a register of tunable product decisions with a recommendation for each.
---

# Product Review

You are the product manager, not the compiler. The question is never "does this
compile" — it is **"is this worth a user's thumb, and is the number right?"**

Before anything else, write down in two lines **what the app is for and who it is
for**, and **the one job a user comes to do** (e.g. "find something happening
tonight → join → show up"). Take it from the README, the store listing, or the
onboarding copy — cite where. Every screen either moves that job along or gets in
the way. If you cannot find the job in the repo, ask the owner once, then proceed.

This skill does four things, in order, and none of them are optional:

1. **Inventory** — what features actually exist, proven from code.
2. **Levers** — every tunable product decision (time windows, caps, gates,
   defaults) surfaced with a recommendation and an owner.
3. **Flows + screens** — reconstruct real user journeys, then score every screen.
4. **Report** — one folder of markdown a non-coder can act on.

## What this is NOT

- Not a code review. Not a security audit → use `security-audit`.
- Not a pixel or accessibility audit → the UI axis **delegates to `ui-ux-review`**, or to
  the project's own design-system doc if it has one. Do not invent a second
  design opinion here.
- Not a rewrite. This skill produces judgements and a ranked action list. It
  changes code only if the owner picks items off that list afterwards.

---

## 0. Safety — this skill is AUDIT ONLY

Non-negotiable. These rules bind you and every subagent you launch.

1. **Read-only on the app.** No edits to source, backend, rules, config or
   dependency files. The **only** files this skill writes are the seven report
   files under `docs/product-review/<date>/`.
2. **No deploys, no git, no releases.** No deploy commands, no release builds, no
   commit, push, branch, merge or PR. Static analysis and the test suite are
   allowed — they only read.
3. **No production data.** The audit reads **code**, not the live database. Do
   not query production, do not use admin credentials or a service account, do
   not read real user records, do not sign in as a user.
4. **Nothing leaves this machine.** Never send code, strings, findings,
   screenshots, logs or user data to any external service. Web lookups are
   allowed only for generic public references, with nothing from the repo in the
   query.
5. **Secrets never enter the report.** If you find a key, token or credential,
   report it as `file:line — secret present in source` and **never copy the
   value**. Deep security work belongs to `security-audit`.
6. **No PII in the report.** Use placeholders like `<user>`, `<item title>`.
7. **Subagents inherit all of it.** Every agent prompt says read-only. Agents
   return text to you; they do not write files or make network calls.
8. **The report stays local** unless the owner explicitly asks otherwise.

If a fix is obvious and tempting, write it in the report as an action row. Do
not apply it. The owner decides what gets changed.

---

## 1. Hard rules

The two failure modes of a product review are **making things up** and **doing
half of it**. Both are silent. Both are fatal.

### Evidence rules (anti-hallucination)

1. **Every claim carries `file:line`.** Features, numbers, flow steps, screen
   scores, copy quotes. A row without a location is **deleted, not softened**.
2. **Numbers come from a command, not from memory.** If you write "58 screens"
   or "the window is 5 hours", the command that produced it goes in
   `05-EVIDENCE.md`. Re-run it; do not trust an earlier session's count.
3. **Never describe a feature you have not opened.** A filename is not evidence
   of behaviour. `admin_reports_screen` existing does not prove reporting works.
4. **Separate PROVEN from JUDGEMENT in every section.** PROVEN = you read the
   code or ran it. JUDGEMENT = your product opinion. Label them. Never let a
   judgement wear a citation's clothes.
5. **No borrowed statistics.** No "users abandon after 3 steps", no invented
   competitor numbers, no benchmark you cannot link. If you need a number you do
   not have, write it as `ASSUMPTION` plus the one metric that would settle it.
6. **A score needs three cited observations** for that screen or feature. Fewer
   → `UNSCORED — not read deeply enough`, logged in the coverage ledger. An
   unscored row is honest; a guessed score is not.
7. **Banned words:** might, could, consider, potentially, seems, appears,
   robust, seamless, leverage, comprehensive. Say what is wrong and what to do.

### Coverage rules (anti-laziness)

8. **Enumerate before you audit.** Get the real file list with the commands in
   §2. Audit every item on it. A buried admin toggle is in scope.
9. **Read screen files whole.** No skimming the first 200 lines and scoring. A
   long file is read in several passes.
10. **The ledger decides when you are done, not a feeling.** `05-EVIDENCE.md`
    carries `screens covered: N/TOTAL` and `features covered: N/TOTAL`.
    Publishing with gaps is allowed; hiding them is not.
11. **If you run out of context, print exactly one line:**
    `TRUNCATED AT [area] — [N] screens / [M] features unchecked`
    Do not pad, do not summarize, do not fake the rest.

---

## 2. Phase 1 — Feature inventory (run the commands)

First identify the stack, then use the matching sweep. Paste the outputs into
`05-EVIDENCE.md` and use the file lists as your coverage checklist.

```bash
# Flutter
find lib -name '*screen*.dart' | sort
grep -rnE 'Navigator\.|context\.(go|push)|GoRoute' lib
grep -n '^exports\.' functions/index.js          # if the backend is Firebase

# React Native / Expo
find app src -name '*.tsx' 2>/dev/null | sort
grep -rnE 'navigation\.navigate|router\.push|<Link' app src

# Next.js / web
find app -name 'page.tsx' 2>/dev/null; find pages -name '*.tsx' 2>/dev/null
grep -rnE 'router\.push|<Link|redirect\(' app src
find app/api pages/api -name '*.ts' 2>/dev/null  # backend entry points

# SwiftUI
grep -rl ': View' --include=*.swift .
grep -rnE 'NavigationLink|navigationDestination|\.sheet\(' --include=*.swift .

# Android / Jetpack Compose
grep -rl '@Composable' --include=*.kt .
grep -rnE 'navController\.navigate|composable\(' --include=*.kt .
```

Adjust the paths to the repo's layout — these are a start, not a contract. If
the stack is not listed, write your own sweep and record it.

For each screen, one inventory row (template in
[references/report-templates.md](references/report-templates.md)): what it is
for, who reaches it and from where, what it writes to the backend, whether it is
finished (`TODO`/stub/dead code counts as unfinished), and whether anything
actually routes to it.

**A screen nothing navigates to is a finding** — either dead weight to cut, or a
feature no user can find.

Then group the rows into **features**. A feature is a user-visible capability,
usually several screens plus a service plus a backend call. Features get scored
in Phase 5.

---

## 3. Phase 2 — Product levers (the part everyone skips)

This is the "should the window be 5 hours?" phase, and it is the highest-value
part of this skill. A **lever** is any number, threshold, window, cap, default or
gate that a human chose and could choose differently.

Hunt them with the recipes in
[references/product-levers.md](references/product-levers.md).

For every lever, fill the register row: current value · every site it is set
(with a **duplicate-site count**) · who owns the decision · what it optimizes ·
what breaks if it is wrong · your recommendation · the one metric that settles
it.

**Two lever findings are automatic on every run:**

- **A lever hardcoded in more than one place is a defect regardless of its
  value.** On the first real run of this skill, one 5-hour window was set at 14
  separate sites across screens, services and providers, and promised to users
  in an FAQ string. Changing 5 → 4 would have been 14 edits plus a copy change,
  with a guaranteed miss. **Recommend one named constant before anyone debates
  the value.**
- **A lever promised in user-facing copy but enforced differently in code is a
  HIGH finding.** Grep the strings for the number, not just the logic.

**Hard gates get the harshest look.** A gate that blocks the core action ("you
cannot join or create anything until you finish X") is the app locking a user out
of its core loop to collect something. State the trade plainly: what it protects,
what it costs, and whether a softer version (nag banner, one-time grace, timed
auto-clear) buys the same protection for less pain.

Levers are the owner's call, not yours. Give a recommendation, name the owner
(Founder / Trust & Safety / Growth / Ops), and stop there.

---

## 4. Phase 3 — Flows

Reconstruct flows **from navigation code**, never from imagination. For each
flow: the ordered screen list with `file:line` for each hop, required user inputs
per step, network waits, and every dead end.

Always map at least these five, renamed to the app's own words:

1. First open → signed in
2. Signed in → profile/setup complete (the onboarding chain)
3. The core action, start to finish (discover → act → confirmed)
4. The main creation flow, if users create anything — count the steps and the
   required fields
5. Trouble: report / block / ban / refund → appeal or recovery

Add any flow the product depends on (checkout, subscription, sharing, the
after-the-event or after-the-purchase loop).

Judge each flow on the only three questions that matter:

- **Length:** how many screens and required fields to first value? Count them.
- **Payoff timing:** does the user see something worth having before the work,
  or only after? Onboarding that asks for six things before showing one result
  is a leak.
- **Dead ends:** any state with no way forward and no explanation. List them.

Verdict per flow: `TOO LONG` / `RIGHT` / `TOO THIN`, with the count that
justifies it and the specific steps to merge, defer, or delete. "Defer" is
usually the right answer — move a step to after first value instead of deleting
it.

---

## 5. Phase 4 — Screen audit (multi-agent shards)

Dozens of screens do not fit one honest pass. Shard the work.

- Sort the screen list from §2. Cut it into shards of **≤ 10 files**.
- Launch **at most 6 agents in flight**, read-only (`subagent_type: Explore` in
  Claude Code), one shard each, using the shard prompt in
  [references/agent-prompts.md](references/agent-prompts.md). Agents return rows
  in the fixed format and nothing else.
- **Agents must not score.** They return cited observations. Scoring happens in
  the main context, once, with the whole picture — so that 1–5 means the same
  thing on screen 3 and screen 53.
- Per-screen lenses live in
  [references/screen-lenses.md](references/screen-lenses.md): first-glance
  clarity, thumb reach, the four states (loading / empty / error / success),
  copy, data honesty, design-system fit, and pressure tests.

If your environment has no subagents, do the shards yourself in sequence — same
format, same rules — and record that in `05-EVIDENCE.md`.

### Phase 4b — the service layer. Do not skip this.

**Screens describe intent. Services describe what actually happens.** A screen
can be individually accurate and still imply the opposite of the system's real
behaviour.

From the first real run of this skill: three onboarding screens each offered a
working Skip button, so the screen pass reported those steps as optional. A
completion service then blocked joining *and* creating unless those exact three
fields were filled. Every screen report was correct; the conclusion drawn from
them was wrong. **Only reading the service caught it**, and it was the most
important finding of the run.

So after the screen shards, read the **gates and the pipeline** in full. Search
for the names gates usually have:

```bash
grep -rnE 'canJoin|canCreate|canPost|canBuy|ensure[A-Z]|isBlocked|isEligible|requires[A-Z]|paywall|[Gg]uard' <src>
```

For each service, capture: what it **blocks** and what the user is shown when
blocked (quote it, or write `SILENT`) · every **silent side effect** — anything
it changes about the user's record that no copy mentions · whether errors
**surface or get swallowed** · and any rule that exists in *only one* place
(client-only or server-only). The prompt is in
[references/agent-prompts.md](references/agent-prompts.md).

Three findings to hunt specifically:

1. **A skip that isn't.** Any step the UI lets you skip, then a gate demands.
2. **A comment that contradicts the code.** E.g. a backend comment says records
   are "deleted immediately by the client", while the client keeps them for
   history and the cleanup job only sweeps one status. Two sources of truth, one
   wrong, data growing forever.
3. **Swallowed errors as a root cause.** If screens show "nothing here" on
   failure, check whether the service ever gave them anything to show. Fix the
   service, not the six screens.

---

## 6. Phase 5 — Verify, then score

**Verify first.** Send every cited row to a fresh verifier agent (prompt in
[references/agent-prompts.md](references/agent-prompts.md)) that re-opens each
`file:line` and returns `CONFIRMED` / `WRONG-LINE` / `NOT-FOUND` / `CLEARED`.

- `NOT-FOUND` → delete the row. No exceptions, no rewording.
- `WRONG-LINE` → fix the citation or delete the row.
- Publish the confirm rate in `05-EVIDENCE.md`. Below 90% means the pass was
  sloppy — re-run that shard before scoring anything.

**Then score.** Six axes, 1–5, defined in
[references/scoring-rubric.md](references/scoring-rubric.md): Clarity, Effort,
Feedback, Fit, Trust, Payoff. Screen score = mean of the six, one decimal.

| Score | Band | What you write |
|-------|------|----------------|
| ≥ 4.0 | **Ship** | polish only — name the one or two small wins |
| 3.0–3.9 | **Fix** | keep the structure, list the named fixes in priority order |
| 2.0–2.9 | **Rework** | the layout or flow is wrong — propose the new arrangement |
| < 2.0 | **Rebuild or cut** | ask whether this screen should exist; if it must, say what replaces it |

Features get four axes (Reachable, Complete, Understandable, Worth-the-code) and
a verdict: `KEEP` / `FIX` / `MERGE` / `CUT`. A `CUT` must state the maintenance
cost — dead features are a tax, not a neutral.

---

## 7. Phase 6 — Write the report

Write to `docs/product-review/YYYY-MM-DD/`. Templates in
[references/report-templates.md](references/report-templates.md). Seven files:

| File | Contents |
|------|----------|
| `00-REVIEW.md` | **the one to read.** Verdict, app product score, top 10 ranked actions with effort × impact, owner questions |
| `01-FEATURES.md` | full inventory + feature scores + KEEP/FIX/MERGE/CUT |
| `02-FLOWS.md` | the flows, step counts, verdicts |
| `03-SCREENS.md` | the scorecard, every screen, sorted worst-first |
| `04-LEVERS.md` | product decision register + recommendations |
| `05-EVIDENCE.md` | commands run, coverage ledger, verify confirm rate, **what was not checked** |
| `06-SERVICES.md` | the gates: what blocks the user, silent side effects, rules enforced in only one place |

Write in plain English: short sentences, name the file and line, say "this is
broken" or "this is fine", give the recommendation instead of a menu.
`00-REVIEW.md` must be readable by someone who does not write code.

---

## 8. Phase 7 — Hand back

End your reply to the owner with, at most:

- The app's product score and a one-line verdict.
- The top 3 actions, in order, with the file to touch.
- The owner-only questions — levers where the trade is a business call, e.g.
  "Chat window: keep at 5 hours, or cut to 3 so chats die while the night is
  still fresh?" Ask these. Do not decide them.

Do not start fixing.

---

## 9. Failure modes of this review

| Failure | What it looks like | Rule that catches it |
|---------|--------------------|----------------------|
| Filename fiction | describing a feature from its name | §1.3 |
| Score inflation | every screen lands 3.5–4.0 | §1.6 + worst-first sort |
| Borrowed stats | "Gen Z abandons after 3 taps" | §1.5 |
| The 12-screen review | 58 screens exist, 12 audited, no ledger | §1.8–1.11 |
| Design double-dipping | inventing UI rules instead of using `ui-ux-review` or the project's design doc | "What this is NOT" |
| Silent lever | noting "5 hours" without asking whether 5 is right | §3 |
| Deciding for the owner | changing a value and calling it a recommendation | §3 last line, §8 |
| Screens-only conclusion | trusting a Skip button without reading the gate | §5 Phase 4b |
