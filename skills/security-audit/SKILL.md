---
name: security-audit
description: Use when auditing code for security vulnerabilities, reviewing auth, permissions or Firestore/Storage rules, before a release or store submission, after a pentest or bug report, when handling secrets, tokens or PII, or when asked "is this secure", "find security holes", "check for vulnerabilities", or "review my rules". Deepest on Flutter + Firebase (Firestore, Storage, Cloud Functions, App Check); the method works on any stack.
---

# Security Audit

## Overview

An audit is **an adversarial proof about a specific attack surface**, not a checklist walk.

A checklist produces "no hardcoded secrets found ✅". An audit produces "signed-in user B can
increment their own `reputationScore` via `PATCH /users/B` because rule line 88 blocks the field
only on `diff()`, and `create` does not check it at all — here is the request, here is the
result." One is a claim. The other is evidence.

**The audit is only as good as its weakest surface.** A perfect review of Firestore rules is
worth little if one unauthenticated callable leaks the same data. So: enumerate the whole
surface first, then prove each piece, then report coverage honestly.

## The Iron Laws

**1. No finding without evidence you personally read.**
Every finding cites `file:line` from a file you opened in this session. Not from memory, not
from a doc in the repo, not from a pattern you assume is there. If you did not read it, it
does not go in the report.

**2. No finding without an exploit path.**
State: *who* (anonymous / any signed-in user / a specific other user / admin), *what they send*,
and *what they get*. If you cannot write that sentence, you have a code smell, not a finding —
file it under Hardening, not under a severity.

**3. No severity without the rubric.**
Severity comes from the table in this skill, not from vibes. "Feels bad" is not High.

**4. Verified or labelled.**
Every finding is `CONFIRMED` (you reproduced it — emulator, test, or the code path is
unambiguous and you traced it end to end) or `PLAUSIBLE` (reasoned but not executed). Never
present PLAUSIBLE as fact. A report with 4 confirmed findings beats one with 30 guesses.

**5. Known non-issues are never reported.**
Check the False Positives table before writing any finding. Reporting Firebase API keys as a
leak destroys the credibility of the other 20 findings.

**6. Coverage is reported, not implied.**
Say what you audited AND what you did not. An unstated gap is a lie by omission.

**Violating the letter of these laws violates the spirit.** "I'm confident it's there" is not
evidence. Go read the line.

## Workflow

Do these in order. Do not start writing findings during step 2 — enumeration first, or you
will rabbit-hole into the first bug and ship a partial audit.

```
1. SCOPE      → what's in, what's out, what changed
2. ENUMERATE  → build the surface ledger (every entry point, every rule path)
3. MODEL      → who attacks this, what do they want
4. PASS       → run the 7 passes against each surface
5. VERIFY     → reproduce findings; promote PLAUSIBLE → CONFIRMED
6. TRIAGE     → severity rubric + false-positive filter
7. REPORT     → findings + coverage ledger + what you could not check
```

### Step 1 — Scope

Ask (or decide and state):
- **Whole app, or the diff?** Diff audits (`git diff main...HEAD`) are cheap and should run
  every branch. Full audits are quarterly / pre-launch.
- **Threat tier?** A pre-launch beta with 50 users is not the same as 50k users with location data.
- **What's out of scope** — third-party SaaS internals, infra you don't control. Say so.

### Step 2 — Enumerate the surface (the ledger)

**Build this before auditing anything.** Every row gets a verdict by the end.

| # | Surface | Entry point | Authenticated? | Verdict |
|---|---------|-------------|----------------|---------|
| 1 | Firestore | `match /users/{userId}` | signed-in | |
| 2 | Callable | `exports.submitReview` | signed-in + App Check | |

Enumeration commands (Flutter + Firebase):

```bash
# Every Cloud Function entry point
grep -aoE "^exports\.[A-Za-z0-9_]+" functions/index.js | sort -u

# Trigger types — onCall/onRequest are attacker-reachable; triggers are not, directly
grep -aoE "(onCall|onRequest|onDocumentCreated|onDocumentUpdated|onDocumentWritten|onDocumentDeleted|onSchedule|onObjectFinalized|beforeUserCreated|beforeUserSignedIn)\(" functions/index.js | sort | uniq -c

# Every rules path
grep -nE "^\s*match /" firestore.rules storage.rules

# Every allow line with its condition
grep -nE "allow (read|write|create|update|delete|list|get)" firestore.rules
```

> `grep -a` matters: a single non-UTF8 byte makes GNU grep treat the file as binary and stop
> after the first match, silently under-reporting your surface. Use `-a`, or the Grep tool.

**Rule: a surface you did not enumerate cannot be reported as secure.**

### Step 3 — Threat model

Skip generic "hackers". Name the actors who actually attack *this* app, and what they want.
The worked example in `references/threat-model.md` is a social/events app; adapt the actors to yours. Short version — the profitable
attacks are not RCE, they are:

- **The reputation launderer** — resets their own strike/no-show counter
- **The ban evader** — keeps using the app after being banned
- **The scraper** — pulls every user profile + location for a rival app or a dataset
- **The free rider** — burns your paid AI/API quota from a script
- **The stalker** — finds one specific person's location or attendance
- **The hijacker** — edits or defaces someone else's event/image

Each maps to a concrete query you should try to write. If you cannot articulate why an actor
*fails*, you have not verified the control.

### Step 4 — The seven passes

Run all seven against each surface. Depth per surface lives in `references/`.

| Pass | Question | Maps to | Playbook |
|------|----------|---------|----------|
| **P1 Authorization** | Can A act as B? Can a user reach an admin path? | OWASP A01, M3, CWE-862 | `firebase-rules.md`, `cloud-functions.md` |
| **P2 Trust boundary** | What does the client control that the server believes? | A06, M8 | `firebase-rules.md` |
| **P3 Input & injection** | Unvalidated input reaching a sink (query, prompt, HTML, shell, email) | A05, M4, LLM01 | `ai-and-content.md` |
| **P4 Secrets & config** | Real secrets in the wrong place; enforcement flags off | A02, M1 | `cloud-functions.md` |
| **P5 Data & privacy** | Who can read what PII; retention; deletion | A04, M6 | `privacy-compliance.md` |
| **P6 Abuse & cost** | Unbounded consumption; no rate limit; billing DoS | LLM10, CWE-770 | `cloud-functions.md` |
| **P7 Integrity & logging** | Tamper detection, audit trail, incident response | A08, A09 | `cloud-functions.md` |

**P1 is where real findings live.** Broken Access Control is #1 in OWASP Top 10 2025 and
Missing Authorization is #4 in the 2025 CWE Top 25. If time is short, do P1 exhaustively and
say you skipped the rest — that beats a shallow pass over all seven.

### Step 5 — Verify

Promote findings from PLAUSIBLE to CONFIRMED. Reading a rule is not verifying a rule.

```bash
# Rules: prove it with the emulator, don't eyeball it
firebase emulators:start --only firestore,auth,storage
npm test --prefix firestore-tests     # @firebase/rules-unit-testing

# Rules compile/lint before anything else
firebase deploy --only firestore:rules --dry-run

# Flutter static analysis
flutter analyze

# Dependency CVEs (M2 / A03 supply chain)
npm audit --prefix functions --omit=dev
flutter pub outdated
```

`@firebase/rules-unit-testing` gives `initializeTestEnvironment`, `assertSucceeds`,
`assertFails`, and is the only supported way to mock auth in rules tests. It talks to the
emulator only and never touches production.

**The verification pattern that finds real bugs:** write the test that *should* fail, and
watch it pass.

```js
// The attacker's request, as a test. If this passes, you have a CONFIRMED finding.
await assertFails(
  bobDb.doc('users/bob').update({ strikeCount: 0 })   // reputation laundering
);
await assertFails(
  bobDb.collection('users').get()                     // mass scrape
);
```

### Step 6 — Triage

**Severity = who can trigger it × what they get.** Find the cell, take the rating.

| Reachable by ↓ / Impact → | Any user's PII / mass data | One other user's data | Integrity / money | Self-only |
|---|---|---|---|---|
| **Anonymous (no auth)** | 🔴 Critical | 🔴 Critical | 🟠 High | 🟡 Medium |
| **Any signed-in user** | 🔴 Critical | 🟠 High | 🟠 High | 🟡 Medium |
| **Specific relationship** (attendee, group member) | 🟠 High | 🟡 Medium | 🟡 Medium | 🟢 Low |
| **Admin/insider only** | 🟡 Medium | 🟡 Medium | 🟢 Low | 🟢 Low |

Then apply modifiers:
- **+1 level** if it is silent (no log, no trace) — an undetectable breach is worse
- **+1 level** if it is scriptable at scale (a loop, not a manual step)
- **−1 level** if a compensating control genuinely blocks it (name the control and the line)
- **Cap at Medium** if exploitation requires a physical device the attacker already owns and
  the blast radius is that device only (M9-class local storage issues)

Anything that cannot be placed in the matrix is **Hardening**, not a vulnerability. Keep those
in a separate section so they never inflate the headline count.

### Step 7 — Report

Use the Output Format below. Include the coverage ledger. Include what you could not verify.

## False Positives — do not report these

Reporting these marks the audit as automated noise. Each is documented as safe by the vendor.

| Common "finding" | Why it is wrong |
|---|---|
| "Firebase API key exposed in client / `firebase_options.dart`" | Firebase API keys **identify**, they do not **authorize**. Google: keys restricted to Firebase services "do not need to be treated as secrets, and it's safe to include them in your code". Protection is Rules + App Check + IAM. |
| "`google-services.json` is in the repo" | Same as above. It contains identifiers, not credentials. (Still fine to gitignore; not a vulnerability.) |
| "Cloud Function bypasses security rules" | The Admin SDK bypasses Rules **by design** — it is a trusted server environment. The finding is only real if the function fails to do its *own* authz check. |
| "No rate limit on Firestore reads from rules" | Rules cannot rate limit. That belongs to App Check, quotas, and budget alerts. |
| "Obfuscation not enabled → critical" | `--obfuscate` raises cost for an attacker; it is not a control. Never Critical on its own. Backend authz is the control. |
| "HTTPS not enforced" (Firebase SDK traffic) | Firebase SDKs use TLS by default. Only real if you found a literal `http://` call. |
| "Secrets in `.env`" | Only a finding if the values are *real secrets* AND the file is tracked by git. Check `git ls-files` before claiming it. Non-sensitive config + feature flags in `.env` is fine. |
| "User enumeration via profile reads" when profiles are intentionally public | Product decision, not a bug. Report the *scale* (mass scrape) instead, which is the real risk. |

**Exception that IS real:** an LLM provider key (e.g. Gemini Developer API key) must never be
in client code or a tracked file — Google documents that one explicitly as a secret.

## Findings that are almost always real

Bias your time toward these. They are where audits of this stack actually pay off.

1. **`allow read: if isAuth()` on a user collection** → any signed-in account scrapes every
   profile. Mitigate with App Check + query constraints (`request.query.limit`), or move
   bulk reads behind a callable. Rules are **not filters** — a `list` rule that could return
   one forbidden doc fails the whole query, so the client query must carry the same
   constraint as the rule.
2. **`create` rules that forget a field `update` blocks.** The `update` rule blocks
   `reputationScore`; does `create` also block it? Attackers create, they don't only update.
3. **Two implementations of one authorization decision.** e.g. rules trust a custom claim
   only, while a Cloud Function *also* accepts a hardcoded UID list. The weaker one wins.
   Grep both layers for the same concept and diff them.
4. **Ban / revoke that doesn't revoke.** Setting `accountStatus: 'banned'` in Firestore does
   not invalidate the attacker's session. Firebase ID tokens live **1 hour, fixed**, and
   `revokeRefreshTokens()` still leaves existing ID tokens valid until they expire. A ban
   without `updateUser(uid, {disabled: true})` + `revokeRefreshTokens(uid)` is a ban the user
   can ignore, and rules that only gate the user's *own* doc leave every other read open.
5. **Custom-claim checks that are stale.** Claims propagate on sign-in / re-auth / token
   refresh (or forced `getIdToken(true)`). A just-granted admin claim is absent from the
   current session's token; a just-revoked one persists up to an hour.
6. **App Check in monitoring mode believed to be enforcing.** A helper that logs on missing
   token but returns `true` is not enforcement. Confirm the flag's *runtime* value, not the
   comment above it. App Check attests the **app/device, never the user** — it is not authz.
7. **Unbounded paid work behind an authenticated endpoint.** Any callable that hits an LLM,
   an image API, or sends email, without a per-user quota, is a billing DoS (LLM10).
8. **Storage `create` with no ownership binding.** A fresh-UUID upload path any signed-in user
   can write is fine only if overwrite/delete are bound to an owner stamp — and if the object
   metadata check handles `null` metadata on pre-existing files.

## Output Format

```markdown
# Security Audit — <scope> — <date>

## Summary
<2–4 sentences: posture, count by severity, the single most important thing to fix.>

Audited: <N> surfaces | Confirmed: <N> | Plausible: <N> | Not audited: <N>

## Coverage Ledger
| # | Surface | Entry point | Verdict |
|---|---------|-------------|---------|
| 1 | Firestore `/users/{id}` | client SDK | 🟠 1 High |
| 2 | Callable `banUser` | signed-in + admin | ✅ Clean |
| 3 | Object storage presigned URLs | — | ⚠️ NOT AUDITED (no access) |

## 🔴 CRITICAL

### C-1 <Title>
**Where:** `functions/index.js:<line>`
**Status:** CONFIRMED (emulator test `bans.test.js:42`)
**Who:** any signed-in user
**Exploit:** <the actual request/steps and what comes back>
**Impact:** <concrete: "reads every user's email + location">
**Why it works:** <the specific line and the wrong assumption>
**Fix:**
```diff
- const isAdmin = request.auth.token.admin === true || ADMIN_UIDS.includes(uid);
+ const isAdmin = request.auth.token.admin === true;
```
**Regression test:** <the assertFails case to add>
**Maps to:** OWASP A01:2025 / CWE-862

## 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
<same shape>

## 🔧 Hardening (not vulnerabilities)
<defence-in-depth items, no severity>

## ✅ Verified Secure
<controls you actively tried to break and could not — name the attack you tried>

## ⚠️ Not Audited / Could Not Verify
<explicit gaps + why + what access would be needed>
```

**"Verified Secure" requires an attack you attempted.** "Rules look fine" is not verification.

## Red Flags — stop and restart the audit

- Writing a finding for a file you have not opened this session
- "This is probably..." / "likely..." / "should be..." in a finding body
- A severity assigned before the exploit path was written
- Reporting a False Positives table entry
- Reporting "no issues found" after auditing < 100% of the ledger
- Fixing code mid-audit (finish the audit; fixes are a separate, reviewable change)
- Skipping the ledger because "the diff is small"

## Rationalizations

| Excuse | Reality |
|---|---|
| "The rules file is 950 lines, I'll spot-check" | Spot-checks miss the one `create` that forgot a field. Enumerate every `match`, then decide what to skip — and say you skipped it. |
| "It's behind auth, so it's low" | "Any signed-in user" is a trivial bar: signup is free. Treat authenticated-reachable as effectively public. |
| "App Check covers it" | App Check attests the app, not the user, and it is off in monitoring mode. It is never an authorization control. |
| "The client never sends that field" | The client is the attacker's tool. Client behaviour is not a control. |
| "Admin SDK bypasses rules so rules don't matter there" | Correct — which is exactly why the function needs its own authz check. |
| "I'll verify with the emulator later" | Then it is PLAUSIBLE, and it says PLAUSIBLE in the report. |
| "That's just theoretical" | Then write the exploit path. If you can't, it's Hardening, not a finding. |
| "The previous audit said this was critical" | Previous audits contain false positives. Re-derive from the vendor docs. |

## References

Deep playbooks — load the one matching the surface you're on:

- `references/threat-model.md` — attacker personas, abuse cases, per-actor test queries
- `references/firebase-rules.md` — Firestore + Storage rules audit, limits, test harness
- `references/cloud-functions.md` — callables, App Check, secrets, quotas, cost DoS
- `references/flutter-client.md` — device storage, obfuscation, deep links, log leakage
- `references/ai-and-content.md` — prompt injection, moderation bypass, output handling
- `references/privacy-compliance.md` — DPDP 2025, GDPR, Play Store requirements
- `references/standards.md` — OWASP/CWE/MASVS mappings with sources and dates

## Remember

- Attackers read your code. Assume the client is hostile and the source is public.
- Least privilege, then defence in depth — one control failing must not be game over.
- The goal is not a long report. It is **a short report you can prove**.
</content>
</invoke>
