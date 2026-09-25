# Standards Map

Every framework below was checked against its primary source. **Dates matter** — citing a
retired list is a credibility hit. Re-verify before quoting in an external report.

Last verified: 2026-08-12.

---

## OWASP Top 10 — 2025 (current)

Released as the 2025 edition, superseding 2021. Source: <https://owasp.org/Top10/2025/>

| ID | Category | Notes for this stack |
|----|----------|----------------------|
| **A01:2025** | Broken Access Control | #1 again. Now absorbs SSRF, privilege escalation, account takeover. **Your Firestore rules live here.** |
| **A02:2025** | Security Misconfiguration | Up from #5 in 2021. Enforcement flags off, permissive rules, debug providers in prod. |
| **A03:2025** | Software Supply Chain Failures | Broadened from "Vulnerable and Outdated Components". `npm audit`, pub deps, CI. |
| **A04:2025** | Cryptographic Failures | PII at rest/in transit, token handling. |
| **A05:2025** | Injection | Includes prompt injection into LLM calls, NoSQL query building. |
| **A06:2025** | Insecure Design | Client-trusted state, missing abuse cases. |
| **A07:2025** | Authentication Failures | Session revocation, ban enforcement, MFA. |
| **A08:2025** | Software or Data Integrity Failures | Unsigned updates, tampered client writes. |
| **A09:2025** | Security Logging and Alerting Failures | Renamed to include **alerting**. Silent breach = worse breach. |
| **A10:2025** | Mishandling of Exceptional Conditions | **New.** Error paths that fail open, leak stack traces, or skip cleanup. |

Two new/renamed categories vs 2021: Software Supply Chain Failures and Mishandling of
Exceptional Conditions. Data basis: contributed application-testing data; A01 appeared in
~3.73% of tested apps across 40 mapped CWEs, A02 in ~3.00% across 16 CWEs.

**Retired framing to avoid:** do not cite "A03:2021 Injection" or "A05:2021 Security
Misconfiguration" numbering as current.

---

## OWASP Mobile Top 10 — 2024 (Final Release)

First major update since 2016. Source: <https://owasp.org/www-project-mobile-top-10/2023-risks/>

| ID | Risk | Flutter/Firebase relevance |
|----|------|----------------------------|
| **M1** | Improper Credential Usage | Hardcoded provider keys, credentials in logs. *Not* Firebase API keys. |
| **M2** | Inadequate Supply Chain Security | pub.dev + npm transitive deps, CI signing. |
| **M3** | Insecure Authentication/Authorization | The big one alongside M9. Ban evasion, stale claims, IDOR. |
| **M4** | Insufficient Input/Output Validation | Rules-level field validation, callable arg validation. |
| **M5** | Insecure Communication | TLS is default in Firebase SDKs; check custom `http://` calls and pinning posture. |
| **M6** | Inadequate Privacy Controls | Location precision, profile exposure, retention, deletion. |
| **M7** | Insufficient Binary Protections | Obfuscation, anti-tamper. Hardening, rarely Critical. |
| **M8** | Security Misconfiguration | Debug App Check provider shipped to prod, emulator hosts left configurable. |
| **M9** | Insecure Data Storage | `SharedPreferences` for anything sensitive; cache of PII. |
| **M10** | Insufficient Cryptography | Home-rolled crypto, weak hashing. |

---

## OWASP MASVS v2 + MASTG

Source: <https://mas.owasp.org/MASVS/>

Eight control groups — use these as the *audit sections* for a mobile-only review:

`MASVS-STORAGE` · `MASVS-CRYPTO` · `MASVS-AUTH` · `MASVS-NETWORK` · `MASVS-PLATFORM` ·
`MASVS-CODE` · `MASVS-RESILIENCE` · `MASVS-PRIVACY`

Companions: **MASTG** (testing guide), **MASWE** (weakness enumeration), **MAS Checklist**.
The checklists were updated in June 2025 to cover all MASTG tests and remapped to the MAS
profiles. `MASVS-STORAGE` was simplified to 2 controls in v2.0.0.

Use MASVS when the deliverable is a *mobile app assessment*; use OWASP Top 10 2025 when it is
*backend/API*. This stack needs both.

---

## OWASP ASVS 5.0.0

Released **30 May 2025** at Global AppSec EU Barcelona. ~350 requirements, three verification
levels (L1/L2/L3), reorganised into 14 chapters — a substantial restructure from 4.0.3, so
requirement IDs are **not** stable across the versions. Never map a 4.x `V2.1.1`-style ID to
5.0 without re-checking.

Use ASVS when you need a formal, level-based assertion ("meets L2"). Overkill for a routine
branch audit.

---

## CWE Top 25 — 2025

Published **11 December 2025** by CISA with MITRE/HSSEDI. Methodology: 39,080 CVE records
published **June 2024 – June 2025**, scored on severity and frequency.

Verified top 5:

| Rank | CWE | Name |
|------|-----|------|
| 1 | CWE-79 | Improper Neutralization of Input During Web Page Generation (XSS) |
| 2 | CWE-89 | SQL Injection |
| 3 | CWE-352 | Cross-Site Request Forgery |
| 4 | **CWE-862** | **Missing Authorization** (up 5 places) |
| 5 | CWE-787 | Out-of-bounds Write |

Also in the top 10: path traversal, use-after-free, out-of-bounds read, OS command injection,
code injection. New entries this cycle include classic/stack/heap buffer overflow, **improper
access control**, **authorization bypass through user-controlled key**, and **allocation of
resources without limits or throttling** (CWE-770).

**For a Flutter/Firebase app**, the memory-safety entries are largely N/A. The ones that bite:
CWE-862 (missing authorization), authorization bypass through user-controlled key (IDOR — the
client supplies the `uid` or `eventId`), and CWE-770 (no quota on an expensive endpoint).

---

## OWASP Top 10 for LLM Applications — 2025

Source: OWASP GenAI project, 2025 edition.

| ID | Risk |
|----|------|
| **LLM01** | Prompt Injection |
| **LLM02** | Sensitive Information Disclosure |
| **LLM03** | Supply Chain Vulnerabilities |
| **LLM04** | Data and Model Poisoning |
| **LLM05** | Improper Output Handling |
| **LLM06** | Excessive Agency |
| **LLM07** | System Prompt Leakage |
| **LLM08** | Vector and Embedding Weaknesses |
| **LLM09** | Misinformation |
| **LLM10** | Unbounded Consumption |

Prompt injection holds #1 for the second consecutive edition, because LLMs process
instructions and data on the same channel with no structural separation. Recommended
mitigations are defence-in-depth: least-privilege tooling, input/output filtering, human
approval for high-risk actions, and regular adversarial testing.

For an app that only does moderation plus text generation (no tools, no retrieval), the live
risks are **LLM01**, **LLM05**, **LLM02** and **LLM10**. LLM06/LLM08 are N/A without agents
or a vector store — say "N/A" rather than padding the report.

---

## Firebase official security checklist

Source: <https://firebase.google.com/support/guides/security-checklist>

Condensed, by category:

**Rules** — start in locked/production mode (deny by default); write rules alongside features,
not after launch; unit test rules with the Local Emulator Suite **and run them in CI**.

**Auth** — mint custom JWTs only server-side; prefer OAuth/OIDC providers; tighten quotas on
`identitytoolkit.googleapis.com`; **enable email enumeration protection**; use Google Cloud
Identity Platform for MFA. Anonymous auth is for temporary onboarding only, and rules must
stop anonymous users reaching non-public data.

**API keys** — restrict keys to Firebase APIs; separate keys for non-Firebase services; FCM
server keys and service-account keys stay secret.

**Functions** — never put sensitive values in environment variables; use **Secret Manager**
for third-party credentials; keep functions simple.

**Abuse / DoS** — monitoring and alerting on Firestore/RTDB/Storage/Hosting; enable App Check
everywhere it is supported; **cap `maxInstances`** to normal traffic; budget alerts; use the
emulator locally to avoid self-DoS.

**Environments** — separate Firebase projects for dev/staging/prod; limit prod IAM access.

**Dependencies** — verify package provenance, read changelogs, scan (e.g. Snyk), monitor after
updates.

---

## Firebase hard limits worth knowing (they cause outages, not just findings)

Source: <https://firebase.google.com/docs/firestore/quotas>

| Limit | Value |
|-------|-------|
| `get()` / `exists()` / `getAfter()` per request | **10** for single-document and query requests; **20** for multi-document reads, transactions, batched writes (applied per operation in a batch) |
| Ruleset source size | 256 KB (console/CLI publish) |
| Compiled ruleset size | 250 KB |
| Nested `match` depth | 10 |
| Path segments in nested matches | 100 |
| Expressions evaluated per request | 1,000 |
| Function call depth | 20 |

Exceeding the document-access limit returns **permission denied** — which looks exactly like a
rules bug. A rules file that leans on `get()` helpers inside `update` conditions can pass unit
tests one at a time and fail in a batched write. Flag helper-heavy rules as a **reliability**
risk, and test them inside a batch.

**Auth token timing** (source: Firebase Admin — manage sessions / custom claims):
- ID tokens expire **1 hour** after minting. Fixed, not configurable.
- Custom claims reach a client on sign-in, re-auth, natural refresh, or forced
  `getIdToken(true)`.
- `revokeRefreshTokens(uid)` kills sessions and blocks new ID tokens, but **already-issued ID
  tokens stay valid until they expire** (up to 1h). Use `verifyIdTokenAndCheckRevoked()`
  server-side when immediate revocation matters.
- Refresh tokens die on user delete, user disable, or a major account change (password/email).

---

## What to cite in a report

Cite the **primary source and its date**, not a vendor blog. Good:

> Firebase API keys are identifiers, not credentials — Google documents that keys restricted to
> Firebase services "do not need to be treated as secrets"
> (<https://firebase.google.com/docs/projects/api-keys>, verified 2026-08-12).

Bad: "Best practice says API keys should never be in source control."
</content>
