# Playbook: Cloud Functions, App Check, Secrets, Cost

Cloud Functions run with the **Admin SDK, which bypasses every security rule**. That is by
design and is not a finding. The finding is always the same shape: *the function did not do the
authorization that rules would have done.*

A callable is a public HTTPS endpoint. Anyone who can read your app bundle knows its name and
its argument shape.

---

## Inventory first

```bash
grep -aoE "^exports\.[A-Za-z0-9_]+" functions/index.js | sort -u
grep -aoE "(onCall|onRequest|onDocumentCreated|onDocumentUpdated|onDocumentWritten|onDocumentDeleted|onSchedule|onObjectFinalized|beforeUserCreated|beforeUserSignedIn)\(" functions/index.js | sort | uniq -c
```

Classify every export. Only two classes are attacker-reachable directly:

| Class | Reachable by | Audit weight |
|-------|--------------|--------------|
| `onCall` | anyone on the internet | **Highest** |
| `onRequest` | anyone, and **no auth is parsed for you** | **Highest** |
| `onDocument*` / `onObjectFinalized` | reachable *indirectly* — attacker controls the document/file that fires it | High (input is hostile) |
| `onSchedule` | not directly reachable | Low, unless it processes attacker-written data |
| `beforeUserCreated` / `beforeUserSignedIn` | runs pre-auth | High (blast radius = signup) |

**Blind spot most audits miss:** background triggers are treated as trusted because "only the
server calls them". Wrong — the *document that fires them* was written by the attacker. A
`onDocumentCreated` on `/joinRequests` processes attacker-authored fields with Admin SDK
privileges. Treat trigger payloads as untrusted input.

---

## The callable checklist

Every `onCall` must, in this order:

```js
exports.doThing = onCall(
  { maxInstances: 5, secrets: ['SOME_KEY'], enforceAppCheck: true },
  async (request) => {
    // 1. Authenticated?
    if (!request.auth) throw new HttpsError('unauthenticated', 'Must be logged in');

    // 2. Authorized for THIS resource? (not just "is logged in")
    //    The client supplies the id — never trust it.
    // 3. Arguments validated: type, length, range, enum
    // 4. Rate limited / quota checked
    // 5. Do the work
    // 6. Audit-log who did what
  }
);
```

Audit each numbered step per function. Missing #2 is the single most common real finding: the
function checks `request.auth` exists, then acts on `request.data.uid` — **authorization bypass
through a user-controlled key** (IDOR), new in the 2025 CWE Top 25.

```js
// FINDING: any signed-in user passes any uid
const { uid } = request.data;
await db.collection('users').doc(uid).update({ ... });

// FIX: derive identity from the token, or check the relationship
const uid = request.auth.uid;
```

### Argument validation

`request.data` is arbitrary JSON from the internet. Check every field:

```js
const { uid, reason } = request.data;
if (!uid || typeof uid !== 'string' || uid.length > 128) {
  throw new HttpsError('invalid-argument', 'uid is required');
}
```
Missing type checks let an attacker pass an object where a string is expected and reach a
different code path. Missing length checks feed unbounded strings into Firestore writes, emails,
and LLM prompts. Flag any `request.data` field used without a type check.

### Admin checks: find all of them and diff them

Grep for every way a caller can become an admin:

```bash
grep -anE "token\.admin|ADMIN_UIDS|OFFICIAL_ADMIN|isAdmin|custom_?claim" functions/index.js
grep -nE "token\.admin|isAdmin" firestore.rules storage.rules
```

**Real pattern worth hunting:** rules were hardened to trust *only* the custom claim, while
functions still accept a hardcoded UID list:

```js
const isAdmin = request.auth.token.admin === true || ADMIN_UIDS.includes(request.auth.uid);
```

Now there are two definitions of "admin" and the weaker one wins. A hardcoded identity also
**cannot be revoked without a deploy**. Report as: inconsistent authorization model, High if
the list is stale or broad, plus a note that revocation is deploy-gated. The legitimate use of
a UID list is a *bootstrap-only* path that mints the claim, never a live authorization check.

### Self-action guards
Admin endpoints need `if (uid === request.auth.uid) throw ...` where relevant (can't ban
yourself), and conversely must not let a non-admin target others.

---

## App Check: what it is and is not

- App Check **attests the app/device**. It does **not** authenticate the user. It is never an
  authorization control. (Google: Firebase Auth protects your users; App Check protects you,
  the developer.)
- Enforcement is supported on Firestore, RTDB, Cloud Storage, **callable functions only** (not
  `onRequest`), Firebase AI Logic, and others.
- `enforceAppCheck: true` makes the platform reject invalid tokens with **401** before your code
  runs. A hand-rolled helper is weaker.

**The failure mode to hunt: monitoring mode mistaken for enforcement.**

```js
const ENFORCE_APP_CHECK = process.env.ENFORCE_APP_CHECK === 'true';

function verifyAppCheck(request, functionName) {
  if (request.app === undefined) {
    if (ENFORCE_APP_CHECK) { throw new HttpsError('failed-precondition', '...'); }
    else { console.warn('App Check MISSING (monitoring mode)'); }  // allows the request
  }
  return true;                                                      // always truthy
}
```

Audit actions:
1. **Read the flag's runtime value**, not the comment. Check `.env`, Secret Manager, and the
   deployed config. A rollback note in a comment means it is *off*.
2. Count call sites vs callable count — an unprotected callable is a hole in the fence:
   ```bash
   grep -ac "verifyAppCheck(" functions/index.js   # protected
   grep -ac "onCall(" functions/index.js           # total
   ```
3. **Flag pre-auth callables explicitly.** A `sendPasswordReset`-style endpoint runs before the
   user is signed in, so App Check is its *only* gate. If enforcement is off, it is an open
   email-sending endpoint — abuse, cost, and a user-enumeration oracle.
4. Verify the client actually produces valid tokens before recommending enforcement: a
   `kDebugMode ? AndroidProvider.debug : AndroidProvider.playIntegrity` split is correct, but
   web platforms skipped at init (`if (!kIsWeb)`) will be **rejected outright** once enforcement
   is on. Enabling enforcement without checking the Verified % in the console is how you take
   your own app down.
5. `request.app === undefined` distinguishes *missing* from *invalid*. An invalid token
   (`{"verifications":{"app":"INVALID"}}`, "Decoding App Check token failed") also needs to be
   rejected — confirm the platform option is doing that, not just the helper.

---

## Secrets

**Rule:** Firebase's own checklist says never put sensitive values in environment variables —
use Secret Manager. `.env` is for non-sensitive config and feature flags.

```bash
# Is anything sensitive tracked?
git ls-files | grep -E "\.env|\.secret|service-account|\.p12|\.jks|\.keystore"
# What's in .env (keys only)
sed -E 's/=.*/=<REDACTED>/' functions/.env
# Declared secrets vs consumed env vars — mismatches are the bug
grep -aoE "secrets: \[[^]]*\]" functions/index.js | sort -u
grep -aoE "process\.env\.[A-Z_]+" functions/index.js | sort -u
```

Findings to look for:
- A `process.env.X` with no matching `secrets: ['X']` on the function → **undefined at runtime**,
  which usually means a silent failure or a fallback code path. Check what happens when the
  value is `undefined` (A10:2025, mishandling exceptional conditions).
- A secret referenced by a function that no longer exists, or vice versa.
- **Never print a secret.** Grep for the secret names inside template literals and log calls.
- A tracked `.env` containing a real key → Critical, and it must be **rotated**, not just
  deleted from HEAD. Git history keeps it.
- **LLM provider keys are genuinely secret** (Google documents the Gemini Developer API key as
  an exception to "Firebase keys aren't secrets"). A Gemini key in client code is Critical.
- Service-account JSON and FCM server keys: always secret.

**Not a finding:** `firebase_options.dart`, `google-services.json`, `GoogleService-Info.plist`.
Those are identifiers. See the False Positives table in SKILL.md.

---

## Session revocation and bans (A07:2025)

Timing facts that decide the severity:
- ID tokens live **exactly 1 hour**, fixed.
- `revokeRefreshTokens(uid)` blocks new tokens but **existing ID tokens stay valid until they
  expire**.
- Refresh tokens die on delete, disable, or major account change.

So a ban implemented as *only* a Firestore write is not a ban:

```js
// Insufficient on its own
await db.collection('users').doc(uid).update({ accountStatus: 'banned' });

// Actually ends the session
await admin.auth().updateUser(uid, { disabled: true });
await admin.auth().revokeRefreshTokens(uid);
// and for immediate effect on your own verification paths:
// await admin.auth().verifyIdToken(token, true)  // checkRevoked
```

Audit questions:
- Is auth disabling **optional** (`if (disableAuth === true)`)? Then the default ban leaves the
  account fully usable. High.
- Are refresh tokens revoked at all? If not, the banned session survives until the user signs
  out.
- Does anything server-side check revocation (`checkRevoked: true`)? If not, accept a ≤1h window
  and **say so in the report** rather than claiming instant revocation.
- Is the ban enforced beyond the user's own document? Rules that only gate `/users/{uid}` leave
  reads and group writes open.

---

## Abuse and cost (LLM10 / CWE-770)

An authenticated attacker with a loop is the realistic threat. Every endpoint that spends money
needs a per-user quota **and** a global ceiling.

| Control | Where | Check |
|---|---|---|
| `maxInstances` | function options | Set on every function? Firebase's checklist calls for capping to normal traffic. |
| Per-user rate limit | Firestore counter, e.g. `rateLimits/{uid}` | Enforced *before* the paid call? Atomic (transaction), or racy read-then-write? |
| Daily global ceiling | aggregate counter | Exists on paid AI/API endpoints? |
| Budget alerts | GCP billing | Configured? |
| Monitoring/alerting | Cloud Monitoring | On Firestore/Storage/Functions? |
| Timeouts | function options | Bounded, so a hung upstream can't pile up instances? |

**Race condition to look for:** a rate limiter that does `get()` → check → `set()` without a
transaction is bypassable by parallel calls. Fire N concurrent requests to confirm; that is a
CONFIRMED finding, not a theoretical one.

**Cost-amplifying chains:** one cheap client call that fans out to an LLM call *plus* an image
API *plus* an email. Rate-limit the entry point, not each leg.

---

## Logging and audit trail (A09:2025)

The 2025 rename adds **alerting** — logs nobody reads are not a control.

- Admin actions (ban, unban, claim grant, appeal review, broadcast) must log actor, target,
  timestamp, reason. Grep the admin callables for a log line naming `request.auth.uid`.
- Never log tokens, secrets, full request bodies, or PII. `console.log` in Cloud Functions goes
  to Cloud Logging, which is broadly readable inside the project.
- Error responses must not leak internals. `throw new HttpsError('internal', 'Failed to ban
  user. Try again.')` while the real error goes to `console.error` is the correct pattern —
  verify the raw `error.message` is not returned to the client.
- Check that failures are *detectable*: a silently swallowed `.catch(() => {})` on a
  security-relevant step (revoking a token, deleting a credential) hides breach evidence. Flag
  each one on a security path.

---

## Supply chain (A03:2025 / M2)

```bash
npm audit --prefix functions --omit=dev
npm ls --prefix functions --depth=0
flutter pub outdated
```
- Pin/verify `firebase-admin` and `firebase-functions` majors; a major bump changes trigger
  APIs and can silently drop options like `enforceAppCheck`.
- Any dependency that reaches the network from inside a privileged function (AWS/S3 clients,
  SMTP via nodemailer, LLM SDKs) is a supply-chain path into your Admin SDK context. Review
  changelogs on update, per Firebase's checklist.
- `node` engine pinned in `package.json` should match a supported runtime.
</content>
