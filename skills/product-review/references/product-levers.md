# Product levers — how to find every decision someone made

A **lever** is a number, threshold, window, cap, default, or gate that a human
chose and could choose differently. They hide as magic numbers in three places:
UI code, service constants, and backend code.

Levers are where product management actually happens. "Should the closing window
be 5 hours?" is a lever question. "Is this widget const?" is not.

---

## 1. The sweep

Run all of these, with `<src>` set to the app source and `<backend>` to the
server code. Paste raw output into `05-EVIDENCE.md`. The patterns are
language-neutral on purpose; add the idioms of your stack (e.g. Dart
`Duration(hours:`, Kotlin `.hours`, Swift `TimeInterval`).

```bash
# Time windows — the richest source
grep -rnE 'Duration\(|\.(hours|minutes|days)|TimeInterval|setTimeout|setInterval|ttl|TTL' <src> <backend>
grep -rnE '[0-9]+ *\* *60 *\* *60' <src> <backend>            # hand-built ms/s windows

# Named caps, floors and thresholds
grep -rnE '(LIMIT|MAX|MIN|WINDOW|CAP|THRESHOLD|COOLDOWN|PCT|BATCH|TARGET|QUOTA)[A-Z_]* *[:=]' <src> <backend>
grep -rnE '(max|min)[A-Z][A-Za-z]* *[:=<>]+ *[0-9]+' <src> <backend> \n  | grep -viE 'width|height|extent|size|lines|scale'   # drop layout noise

# Input limits the user feels
grep -rnE 'maxLength|maxLines|max_length|maxlength' <src>

# Query limits — these silently define "how much of the app exists"
grep -rnE '\.limit\(|LIMIT [0-9]+|pageSize|per_page' <src> <backend>

# Gates: what blocks the core loop
grep -rnE 'canJoin|canCreate|canPost|isBlocked|isEligible|accountStatus|profileComplete|paywall' <src> <backend>

# Retention / deletion clocks
grep -rniE 'deletionScheduled|autoDelete|retention|expir|archive|purge' <src> <backend>
```

Then read the UI defaults by hand — pre-filled fields, initial toggle values, the
starting sort order. Grep cannot tell a default from any other assignment.

---

## 2. Lever families, and the question to ask each

### Time windows

Content lifetime, read-only delays, auto-archive, request expiry, cache TTL,
cooldowns, deletion grace periods.

Ask: **does this match how the user's real life works?** Windows should map to
human time, not round numbers. Cite the user-facing copy that promises the
window; a mismatch between copy and code is a HIGH finding.

### Caps and floors

Group sizes, age ranges, message length, image count, posts per day, strike
caps, feed page size.

Ask: **who does this cap exclude, and did we mean to?** A default group size of
4 makes a social app an intimate-hangout app; 40 makes it a party app. That is a
positioning decision hiding in a form default. An age default of 18–25 quietly
narrows every new item's audience. Cite the default's `file:line`.

### Hard gates

Anything that blocks joining, creating, or messaging.

Ask: **is a lockout the smallest tool that works?** For each gate, write: what
it protects · what a blocked honest user experiences · the softest alternative
(banner, one-time grace, timed auto-clear, degraded mode). A gate with no
visible explanation on the blocked screen is a Trust-axis 1.

Typical shape: a flag like `canJoinOrCreate` that goes false while the user has
some pending obligation (an unfinished profile, an unrated past event, an unpaid
invoice). Find it, and find what the blocked user is told.

### Defaults

Pre-filled values, pre-selected toggles, notification opt-ins, sort order.

Ask: **is the default the majority case, or the lazy case?** Defaults are the
strongest product lever that costs nothing to change. Most users never move
them. Every default is a decision you made for the whole user base.

### Rate limits and moderation thresholds

Strike windows, report limits, ban durations, appeal cooldowns.

Ask: **what happens to a false positive?** Cite the appeal path. If the code
punishes but the app has no route to appeal on that screen, that is a finding
regardless of the threshold.

### Copy promises

Any number stated in a user-facing string: FAQ answers, empty states, toasts,
onboarding text.

Ask: **does the code do what the string says?** Grep the string, then verify the
enforcement site.

---

## 3. Register row format

One row per lever in `04-LEVERS.md`:

| Field | Rule |
|-------|------|
| **Lever** | plain-English name, not the variable name |
| **Current value** | as enforced in code |
| **Set at** | every `file:line`, plus the count. More than one site = defect. |
| **Promised at** | user-facing copy sites, or `none` |
| **Owner** | Founder / Trust & Safety / Growth / Ops |
| **Optimizes for** | the one thing this value protects |
| **Breaks if wrong** | too-low failure and too-high failure, both named |
| **Recommendation** | split into PROVEN (structural, e.g. "extract a constant") and JUDGEMENT (the value itself) |
| **Settles it** | the single metric or experiment that answers it |

## 4. Rules for recommending a value change

1. **Structure before value.** If a lever is duplicated, recommend extracting
   one constant first. Debating 5 vs 3 hours while the number lives in 14 places
   is theatre.
2. **Never change a lever inside this skill.** The review recommends; the owner
   decides; a later task edits.
3. **Every value recommendation carries the metric that would prove it wrong.**
   No metric, no recommendation — downgrade it to an owner question.
4. **If a lever is fine, say "keep it" and move on.** A register full of
   invented changes is worse than a short one. Levers that are correct should be
   listed as correct, so nobody re-opens them next quarter.
5. **Never claim a lever was A/B tested, requested by users, or measured**
   unless you can cite where. Absence of data is itself a finding: write
   `NO DATA — instrument this first`.
