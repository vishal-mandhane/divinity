# Playbook: Firestore + Storage Security Rules

Rules are the **only** thing between a signed-in client and your database. Cloud Functions
using the Admin SDK bypass them entirely by design, so rules govern exactly one thing: what the
Flutter client can do with a stolen or scripted session.

Assume the attacker has: a valid account, your app's API key, your collection names, every
document ID visible in any UI, and the ability to send arbitrary REST calls. Nothing in the
client is a control.

---

## Order of work

1. **Inventory** every `match` block and every `allow`.
2. **Classify** each path: public-read, owner-only, relationship-gated, server-only.
3. **Diff create vs update vs delete** on every path — this is where the bugs are.
4. **Trace helper functions** to their actual definition; do not trust their names.
5. **Attack** each path with the emulator test harness.
6. **Check the fail-closed default.**

```bash
grep -nE "^\s*match /" firestore.rules storage.rules
grep -nE "allow (read|write|create|update|delete|list|get)" firestore.rules
grep -nE "^\s*function " firestore.rules
```

---

## The 12 rule-level defects that matter

### 1. `allow read: if isAuth()` on anything with PII
Any signed-in account can read the whole collection. Signup is free, so this is
**effectively public**. Rate this by the data, not by the auth requirement.

Real shape:
```javascript
match /users/{userId} {
  allow read: if isAuth();   // every profile: email, age, location, interests
}
```
Mitigations, in order of strength: move bulk reads behind a callable that returns only public
fields → split private fields into a `private/` subcollection → constrain `list` with
`request.query.limit` → App Check (raises cost, does not fix authz).

### 2. `create` forgets what `update` blocks
The classic. An `update` rule carefully blocks `reputationScore`, `bannedAt`, `banReason` via
`diff().affectedKeys().hasAny([...])` — and the `create` rule never mentions them. The attacker
deletes their doc and re-creates it with forged values, or creates it pre-forged on signup.

**Audit action:** for every field the `update` rule protects, grep the `create` rule for the
same field. Build a two-column table. Any field protected on one side and not the other is a
finding.

```javascript
// update blocks it...
allow update: if isOwner(userId)
  && !request.resource.data.diff(resource.data).affectedKeys()
       .hasAny(['reputationScore', 'bannedAt', 'bannedBy', 'banReason']);
// ...does create? If `allow create: if isOwner(userId);` — no. Finding.
```

### 3. `diff()` only sees *changed* keys
`diff(resource.data).affectedKeys()` is a delta. A field written with the *same* value is not
in the set. Rules that rely on "the key didn't change" are fine; rules that rely on "the key
was never present" are not. Also: on `create` there is no `resource.data`, so `diff()` cannot
be used at all — a copy-pasted `update` condition silently breaks or throws on create.

### 4. Monotonic counters that aren't
```javascript
&& request.resource.data.get('strikeCount', 0) >= resource.data.get('strikeCount', 0)
```
This blocks a decrease — good. But it permits an *unbounded increase*, and it permits the
client to set it on `create`. If a counter drives a reward, allow only `+1`:
```javascript
&& request.resource.data.strikeCount == resource.data.strikeCount + 1
```
Same class: "only one entry removed at a time" checks on arrays compare `.size()` only, not
*which* entry. A client can remove the wrong element and add another, keeping size constant.
If identity matters, compare content, not length.

### 5. Server-owned fields writable by the client
Any field a Cloud Function computes (reputation, verification status, `isFeatured`,
`isOfficial`, `status`, `snapshotCreated`) must be unwritable from the client. Grep the rules
for each one. If the rules don't name it, the client can set it.

### 6. Ban / suspension enforced only on the user's own document
```javascript
&& resource.data.get('accountStatus', 'active') != 'banned'
```
Stops a banned user editing *their own* doc. Does nothing about their reads of `/events`,
`/users`, or their writes to `/groups`. A ban must be enforced at the **session** layer
(`disabled: true` + `revokeRefreshTokens`), not per-collection. See `cloud-functions.md`.

### 7. Relationship checks that read a document the attacker controls
```javascript
function isGroupMember(eventId) {
  return request.auth.uid in groupDoc(eventId).members;
}
```
Secure only if `members` is itself unwritable by a non-member. If any client can add itself to
`groups/{id}.members`, every rule built on `isGroupMember` collapses. **Always trace the
gate's data source back to its own write rule.** A relationship gate is exactly as strong as
the weakest write path into the document it reads.

### 8. Rules are not filters (the `list` trap)
Firestore evaluates a query against its **potential** result set. If the query *could* return a
document the rule forbids, the whole request fails — rules never silently filter. So:

```javascript
allow read: if request.auth.uid == resource.data.author;
```
```js
db.collection("stories").get();                              // fails
db.collection("stories").where("author","==",uid).get();      // succeeds
```
Two consequences for the audit:
- A path with owner-only `read` and no client-side `where` is a **broken feature**, not a
  secure one. Check the client actually constrains the query.
- Splitting `read` into `get` and `list` is the fix when single-doc reads should be broader
  than collection scans:
  ```javascript
  allow get: if isAuth();
  allow list: if isAuth() && request.query.limit <= 50;
  ```

### 9. Storage: `create` with no ownership binding
A path where any signed-in user may write a fresh UUID is acceptable *only* if
overwrite/delete are bound to an owner. The UUID is **not a secret** — it is embedded in the
document's `imageUrl`, which every signed-in user can read.

Correct shape (split create from overwrite):
```javascript
// create: fresh object only
allow write: if request.auth != null
  && resource == null
  && request.resource.size < 10 * 1024 * 1024
  && request.resource.contentType.matches('image/.*');

// overwrite/delete: only the stamped owner
allow write: if request.auth != null
  && resource != null
  && (resource.metadata == null
      || resource.metadata.get('ownerId', '') == ''
      || resource.metadata.get('ownerId', '') == request.auth.uid);
```
Two things to verify, both easy to get wrong:
- **`request.resource == null` on delete.** Size/content-type conditions must be guarded or
  deletes break.
- **`resource.metadata` may be null**, not an empty map, on objects written before the stamp
  existed. Null-check first and rely on `&&` short-circuit, or every legacy-file cleanup starts
  failing. Note the legacy-tolerant clause is a *deliberate, shrinking* exception — flag it as
  a tracked risk, not a clean pass.
- **`{fileName}` binds one path segment.** `event_images/{fileName}` does **not** cover
  `event_images/thumbnails/x.jpg`. A missing sibling block means the default-deny catches it
  (feature break) or, worse, a `{allPaths=**}` block earlier grants it.

### 10. `contentType` is client-supplied
`request.resource.contentType.matches('image/.*')` checks a header the uploader chose. It stops
accidents, not attacks. Real image validation is server-side (a Storage trigger that inspects
magic bytes / re-encodes). Report a missing size cap as the real finding — that one is
enforceable and it is your bill.

### 11. Helper-heavy rules and the `get()` budget
`get()`/`exists()`/`getAfter()` are capped at **10** per single-document or query request and
**20** per multi-document read, transaction, or batched write — applied per operation in a
batch. Every `eventDoc()`/`groupDoc()` call inside a condition spends budget, and each is a
**billed read**. Rules that pass unit tests individually can throw permission-denied inside a
batch. Also: nested `match` depth 10, max 1,000 expressions per request, function call depth
20, 256 KB source / 250 KB compiled.

Audit action: count `get(` calls on the worst-case path of the most complex rule, then write a
batched-write test.

### 12. Default-deny and shadowing
Firestore: an unmatched path is denied — but a broad `match /{document=**}` with a permissive
`allow` earlier in the file grants everything beneath it. Storage: confirm the
`match /{allPaths=**} { allow read, write: if false; }` default exists and that no later block
widens it unintentionally. In Firestore, **`allow` statements are additive** — a second
`allow write` on the same path is an OR, not an override. Two `allow write` blocks means the
*weaker* one wins. Read every path's full set of allows together.

---

## Test harness

`@firebase/rules-unit-testing` is the only supported way to mock auth in rules tests, and it
only ever talks to the emulator.

```js
const { initializeTestEnvironment, assertFails, assertSucceeds }
  = require('@firebase/rules-unit-testing');
const fs = require('fs');

let env;
beforeAll(async () => {
  env = await initializeTestEnvironment({
    projectId: 'demo-app',
    firestore: { rules: fs.readFileSync('firestore.rules', 'utf8') },
  });
});
afterAll(() => env.cleanup());
beforeEach(() => env.clearFirestore());

const alice = () => env.authenticatedContext('alice').firestore();
const bob   = () => env.authenticatedContext('bob').firestore();
const anon  = () => env.unauthenticatedContext().firestore();
// custom claims:
const admin = () => env.authenticatedContext('root', { admin: true }).firestore();

// Seed data that rules would otherwise block:
await env.withSecurityRulesDisabled(async (ctx) => {
  await ctx.firestore().doc('users/alice').set({ strikeCount: 3 });
});
```

**Write the attacker's test, not the happy path.** Minimum set per collection:

```js
test('anon is locked out',        () => assertFails(anon().doc('users/alice').get()));
test('no cross-user write',       () => assertFails(bob().doc('users/alice').update({bio:'x'})));
test('no reputation laundering',  () => assertFails(bob().doc('users/bob').update({strikeCount:0})));
test('no forging on create',      () => assertFails(bob().doc('users/bob').set({reputationScore:99})));
test('no self-unban',             () => assertFails(bob().doc('users/bob').update({accountStatus:'active'})));
test('no mass scrape',            () => assertFails(bob().collection('users').get()));
test('no server field write',     () => assertFails(bob().doc('events/e1').update({isOfficial:true})));
test('owner still works',         () => assertSucceeds(alice().doc('users/alice').update({bio:'ok'})));
test('batch respects get budget', () => { /* batched write over the helper-heavy path */ });
```

Every finding you fix gets a permanent `assertFails` here. **A fix without a regression test
is not a fix** — the same hole comes back on the next refactor.

Run in CI. Firebase's own checklist calls for rules unit tests in the CI pipeline.

```bash
firebase emulators:exec --only firestore,auth,storage "npm test"
```

---

## Reporting rules findings

Quote the rule, name the wrong assumption, give the diff. Do not say "rules are too
permissive".

```markdown
**Where:** `firestore.rules:84-128` (`allow update` on `/users/{userId}`)
**Who:** any signed-in user, on their own document
**Exploit:** delete own doc, re-create with `{reputationScore: 9999}` — `allow create: if
isOwner(userId)` performs no field validation, while `update` blocks the field at line 88.
**Impact:** unlimited forged reputation; feeds the rewards the server grants from this field.
**Fix:** extract the field guard into a shared function and apply it to both create and update.
**Regression test:** `assertFails(bob().doc('users/bob').set({reputationScore: 99}))`
**Maps to:** OWASP A01:2025, CWE-862
```
</content>
