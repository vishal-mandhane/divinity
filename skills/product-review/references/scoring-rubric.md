# Scoring rubric

Two scales: one for **screens**, one for **features**. Both are 1–5 integers.
No half points — half points are how score inflation starts.

A score is only allowed when you have **three cited observations** for that
row. Otherwise write `UNSCORED — not read deeply enough` and log it in the
coverage ledger.

---

## Screen scale — six axes

### 1. Clarity — "in 3 seconds, do I know what this screen is for?"

| | |
|--|--|
| **5** | One headline, one obvious next action. A first-timer names the screen's job correctly. |
| **4** | Clear after a beat. One competing element. |
| **3** | Two things compete for attention. The user reads to figure it out. |
| **2** | No clear primary action. Three or more equal-weight options. |
| **1** | The user cannot tell what they are looking at, or the title lies about the content. |

Evidence to cite: the widget tree order, the title string, how many buttons sit
above the fold, `AppBar` actions count.

### 2. Effort — "how much work to finish the job?"

| | |
|--|--|
| **5** | One tap, or one field, and it is done. |
| **4** | 2–3 taps, no typing, or one short field. |
| **3** | Under 5 inputs, all of them genuinely needed. |
| **2** | 6+ inputs, or a required field the app could infer (city, age from DOB, country from locale). |
| **1** | A required step with no path forward, or the user must leave and come back to continue. |

Evidence to cite: count of `TextField`/`TextFormField`, required validators,
step count in multi-step screens, `Navigator` hops needed.

**Anything the app can infer but asks for anyway costs a point.** Location,
country and age are the usual offenders.

### 3. Feedback — the four states

Score by how many of loading / empty / error / success are handled **on this
screen**:

| | |
|--|--|
| **5** | All four, and each one is specific ("Nothing near you tonight — start one?"). |
| **4** | All four, one is generic. |
| **3** | Three of four. |
| **2** | Two of four — usually loading and success, with error swallowed. |
| **1** | The user taps and nothing visibly happens, or a failure is silent. |

Evidence to cite: `isLoading` branches, empty-state widget, `catch` blocks that
show something to the user, a toast or snackbar on success. A `catch` that only
writes to a log is **not** user feedback — score it as missing.

### 4. Fit — brand and design system

Do **not** invent design rules here. Run the project's own design-system
checklist if it has one, otherwise the `ui-ux-review` skill, and convert
violations to a score:

| | |
|--|--|
| **5** | Zero violations. Light and dark themes both handled. |
| **4** | 1–2 cosmetic violations (a magic number, one stray radius). |
| **3** | 3–5 violations, or one missing dark-mode branch. |
| **2** | A pattern the design system bans is present (e.g. decorative gradient, shadow soup, off-brand font). |
| **1** | The screen ignores the theme entirely — hardcoded colours and fonts throughout. |

Cite the violation lines. Do not re-litigate the design system's rules.

### 5. Trust — does the user understand the consequences?

Most apps can flag, block, ban, charge or delete. This axis is about whether
the user knows what is about to happen to them.

| | |
|--|--|
| **5** | Consequence stated in plain words before the action, with a way out and a way to appeal. |
| **4** | Stated, but the wording is soft or the reversal path is buried. |
| **3** | A confirm dialog exists, but it does not say what actually happens. |
| **2** | Irreversible or punitive action with no confirmation. |
| **1** | The user is punished or locked out and the screen does not say why, or offers no route to appeal. |

Cite the confirm dialog, the copy string, the appeal/route-out widget.

### 6. Payoff — "was it worth my thumb?"

| | |
|--|--|
| **5** | The user gets something they wanted immediately after acting. |
| **4** | Clear payoff, one step delayed. |
| **3** | Payoff exists but is invisible on this screen (the user must go looking). |
| **2** | The user does work now for a benefit later that the screen never names. |
| **1** | Pure extraction: the app takes input and gives nothing back. |

Onboarding screens fail this axis most often. Asking for interests, a bio and a
photo before the user has seen a single result is a 1 or 2, no matter how nice
it looks.

---

## Screen score and bands

`score = mean(six axes)`, one decimal. Sort the scorecard **worst first** — it
stops the report from reading like a victory lap.

| Score | Band | Required output |
|-------|------|-----------------|
| ≥ 4.0 | Ship | 1–2 polish wins, named |
| 3.0–3.9 | Fix | ordered list of named fixes, structure kept |
| 2.0–2.9 | Rework | propose the new arrangement, not a list of tweaks |
| < 2.0 | Rebuild or cut | argue whether the screen should exist; if yes, what replaces it |

**Hard override:** any axis scoring 1 caps the screen at **2.9** regardless of
the mean. One broken fundamental is not averaged away by four nice ones.

**App product score** = mean of all screen scores, weighted:
core-loop screens (every screen the one job from SKILL.md passes through) count
**×3**; onboarding and auth **×2**; admin and settings **×1**. State the weights
in the report so the number is reproducible.

---

## Feature scale — four axes

| Axis | 1 | 5 |
|------|---|---|
| **Reachable** | nothing navigates to it | reachable in ≤ 2 taps from the place users already are |
| **Complete** | stub, `TODO`, or half-wired (UI with no write path, or write path with no UI) | end-to-end: UI → service → backend → back to UI |
| **Understandable** | users cannot tell it exists or what it does | self-explanatory, or explained where it is used |
| **Worth-the-code** | high maintenance, near-zero user value | small surface, central to the core loop |

Verdict per feature:

- **KEEP** — mean ≥ 4.0
- **FIX** — mean 3.0–3.9, or any single axis at 2
- **MERGE** — duplicates another feature's job (cite both)
- **CUT** — mean < 3.0. **State the maintenance cost**: files, lines, functions,
  database tables/collections and indexes it keeps alive. A dead feature is a tax.

---

## Worked example — a lever, fully filled in

From a real run of this skill on a social events app. Paths shortened here; the
shape is what to copy.

> **Lever:** event lifetime / chat closing window = **5 hours**
> **Set at:** 14 sites — a status calculator (×3), the group chat screen (×3),
> the messages screen (×2), a status service (×2), two providers, the "my
> events" screen and a privacy settings screen
> **Promised to users at:** one FAQ answer string
> **Owner:** Founder (it defines how long a night on the app lasts)
> **Optimizes for:** keeping chat alive long enough to coordinate arrival and
> after-plans
> **Breaks if wrong:** too short → the group dies mid-event and people cannot
> regroup. Too long → the feed shows finished events as "ongoing", and archives
> fill with dead chats.
> **Recommendation (PROVEN part):** extract one constant before touching the
> value. 14 duplicated sites plus a copy string means any change is a 15-edit
> job with a guaranteed miss.
> **Recommendation (JUDGEMENT part):** keep 5 hours for now. It matches a
> real night out.
> **Settles it:** median time from event start to last message in the group
> chat. If p90 lands under 3 hours, the window is too long.

In your report, every site gets its real `file:line`, and every lever row gets a
PROVEN half and a JUDGEMENT half.
