# Playbook: Privacy & Compliance

Maps to OWASP Mobile M6 (Inadequate Privacy Controls), A04:2025, and MASVS-PRIVACY.

Privacy findings are the ones that become **store rejections, regulator letters, and press** —
not just tickets. Treat "we collect precise location and never say so" as seriously as an authz
bug.

Verified 2026-08-12. Regulations move; re-check dates before citing externally. **This is
engineering guidance for an audit, not legal advice** — flag obligations, recommend counsel for
interpretation.

---

## 1. Data inventory (do this first — everything else depends on it)

You cannot audit privacy without knowing what is stored. Build the table from the **schema**,
not from the privacy policy.

| Field | Collection | Sensitivity | Who can read | Disclosed? | Deleted on account deletion? |
|---|---|---|---|---|---|
| `email` | `users` | PII | any signed-in user | ? | ? |
| `latitude`/`longitude` | `events` | **precise location** | any signed-in user | ? | ? |
| `age` / `dateOfBirth` | `users` | PII | any signed-in user | ? | ? |
| `fcmToken` | `users/private` | device id | owner only | ? | ? |

Three columns are the audit:
- **Who can read** — from the rules, not from the UI.
- **Disclosed** — matches the privacy policy and the Play Data safety form.
- **Deleted** — actually removed by the deletion path.

Any row where "who can read" is broader than what the UI shows is an **undisclosed exposure**.

---

## 2. Location precision (the highest-harm item in a social app)

- If the UI shows a city, does the backend store and serve **exact coordinates**? Any signed-in
  user reading exact lat/lng for a person's events is a physical-safety issue.
- Recommend: store coarse/truncated coordinates for display, keep precise values server-side
  only where genuinely required, and never expose another user's precise location through rules.
- Google Play requires accurate **precise vs approximate** location disclosure in the Data
  safety section — a mismatch between what you store and what you declare is a policy violation
  on top of the privacy risk.
- Check the runtime permission asked for (fine vs coarse) matches the actual need.

---

## 3. Account deletion (Google Play requirement)

Google Play's User data policy: **if your app lets users create an account, it must let them
request account deletion both in-app and through a web resource**, and deleting the account must
delete the associated user data (limited retention is allowed for security, fraud prevention, or
legal/regulatory reasons — and should be documented).

Audit the deletion path against the **full schema**, not against the code's own comments:

```bash
grep -rlnE "deleteAccount|deleteUser|account_?deletion" lib/ functions/
grep -anE "cleanup|deleteUser|accountStatus" functions/index.js | head -30
```

Checklist:
- [ ] In-app deletion entry point exists and is discoverable
- [ ] **Web deletion route exists** (this is the one teams forget)
- [ ] `users/{uid}` + every subcollection (`private`, `favorites`, `kickedEvents`, …)
- [ ] Firebase Auth user deleted or disabled
- [ ] Storage objects: profile photo, event images, thumbnails, appeal images, bug screenshots
- [ ] FCM tokens removed (stop push to a deleted account)
- [ ] Content authored by the user: events, requests, group memberships, votes, notifications
- [ ] Cached copies (denormalised `creatorName`/`creatorPhotoUrl` on other documents)
- [ ] Grace period behaviour: during `pending_deletion`, is the account still fully usable?
- [ ] Scheduled cleanup actually runs and is monitored (a silently failing scheduled function
      means data is retained past the promise)
- [ ] Retention exceptions are documented and justified

**Denormalised copies are the classic gap.** If `creatorName` and `creatorPhotoUrl` are stamped
onto every event, deleting the user leaves their name and photo URL visible across the app.

---

## 4. India DPDP (relevant if you have Indian users)

The Digital Personal Data Protection Rules, 2025 were **notified 13 November 2025**, with a
phased runway: most operational obligations (notice and consent, breach notification, data
principal rights) become enforceable across an **18-month** implementation window, with full
compliance required by around **13 May 2027**. Consent manager provisions take effect **12
months** from notification (around 13 November 2026).

What an audit should check now:
- **Notice**: clear, itemised purpose, categories of data, retention period, and how to
  withdraw consent.
- **Consent**: informed, unambiguous, freely given, purpose-specific — not bundled into a
  blanket ToS acceptance. Withdrawal must be as easy as granting.
- **Rights mechanisms**: access, correction, erasure, grievance redressal — with a named contact
  and a working channel.
- **Breach notification**: a documented process that can actually be executed (who detects, who
  notifies, in what window). Ties directly to A09:2025 — if you have no alerting, you cannot
  notify.
- **Children's data**: DPDP treats under-18s as children requiring verifiable parental consent.
  An 18+ app must therefore have a **real** age gate, and the age value must not be
  client-forgeable — check the rules validate `minAge`/`age` and that nothing lets a user edit
  their age freely after signup.
- **Retention limits**: define and enforce them; unbounded retention is a finding.

---

## 5. GDPR / general

If you have EU users: lawful basis, DSARs (access/portability/erasure), data minimisation,
purpose limitation, records of processing, and breach notification (72h).

- **Data export** (portability): if an `exportUserData` endpoint exists, audit it as a security
  surface too — it is a single call that returns everything about a user. It must verify identity
  strictly, rate limit hard, and return only the caller's own data.
- **Data minimisation** as an audit question: is every collected field actually used? Unused PII
  is pure liability — recommend deletion of the field, not better protection of it.
- Third-party processors (moderation APIs, image search, email, analytics, object storage) are
  sub-processors: they must be disclosed, and you should know what you send them. Grep the
  proxy functions for exactly which user data crosses the boundary.

---

## 6. Google Play / App Store operational requirements

- **Data safety section** must match reality — including precise vs approximate location, and
  whether data is shared with third parties. Audit it against your data inventory table.
- **Target API level**: existing apps must target **Android 15 (API 35) or higher by 31 August
  2026**; apps below that stop being discoverable on newer Android versions.
- **Privacy policy** must be linked and current, and must actually describe the third parties
  you send data to.
- Permissions must be justified; sensitive permissions need declarations.

---

## 7. Reporting privacy findings

Anchor to the concrete exposure, then the obligation:

```markdown
**Where:** `firestore.rules:209` — `allow read: if isAuth()` on `/events/{eventId}`
**Who:** any signed-in user
**Exposure:** `latitude`/`longitude` are precise coordinates; the UI only displays city. Any
account can read the exact location of every event, including private/small gatherings, and
correlate them with `attendees` to locate a specific person.
**Obligation:** Play Data safety requires accurate precise-location disclosure; DPDP requires
purpose-limited processing with notice.
**Fix:** serve truncated coordinates to non-attendees; restrict precise values to the creator
and confirmed attendees.
**Maps to:** OWASP M6, A01:2025
```
</content>
