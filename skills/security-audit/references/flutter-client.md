# Playbook: Flutter Client

**Framing that keeps you honest:** the client is the attacker's machine. Nothing here is a
security control — it is damage limitation for a lost/rooted device and cost-raising against
reverse engineering. So client findings are rarely Critical, and a report that leads with
"obfuscation not enabled" while the Firestore rules leak every profile has its priorities
inverted.

Maps to OWASP Mobile Top 10 2024 M1/M7/M8/M9 and MASVS-STORAGE / -RESILIENCE / -PLATFORM.

---

## What actually deserves Critical/High on the client

| Finding | Severity | Why |
|---|---|---|
| A real third-party secret compiled into the app (LLM key, SMTP creds, private API key) | 🔴 Critical | Extractable from the bundle in minutes; costs money and can't be rotated without a release |
| PII or tokens written to plaintext local storage on a shared/rooted device | 🟡 Medium | Blast radius is that device (M9) |
| Sensitive data in logs shipped off-device | 🟠 High | Aggregated across all users |
| Auth state trusted from local storage for an authorization decision | 🟠 High | Client-side gate = no gate |
| No obfuscation | 🟢 Low / Hardening | Raises cost only |
| Firebase API key in `firebase_options.dart` | **not a finding** | Identifier, not credential |

---

## 1. Secrets in the bundle (M1)

```bash
# Real secrets, not Firebase identifiers
grep -rnE "(sk-[A-Za-z0-9]{16,}|AIza[0-9A-Za-z_-]{35}|xox[baprs]-|-----BEGIN [A-Z ]*PRIVATE KEY)" lib/ --include="*.dart"
grep -rniE "(api_?secret|client_?secret|smtp_?pass|private_?key|bearer )" lib/ --include="*.dart"
```
Triage each hit:
- `AIza…` in `firebase_options.dart` → **not a finding** (Firebase identifier).
- `AIza…` that is a **Gemini/Generative Language** key → **Critical**. Google documents this one
  as a genuine secret. It belongs in Secret Manager behind a callable proxy.
- Any third-party key that isn't scoped to a domain/bundle → Critical, rotate.

**The correct architecture, and what to check for:** the client calls a callable proxy
(e.g. `geocodeProxy`, `imageSearchProxy`); the key lives in Secret Manager; the proxy
enforces auth + App Check + rate limit. If you see proxy functions, confirm the client no longer
holds the key *and* that the proxy actually validates. A proxy without a rate limit just moves
the cost problem server-side.

## 2. Local storage (M9 / MASVS-STORAGE)

```bash
grep -rn "SharedPreferences\|Hive\|sqflite\|FlutterSecureStorage\|path_provider" lib/ --include="*.dart" | head -40
```
Classify what is stored:
- **Fine in SharedPreferences:** UI prefs, feature flags, cache of already-public data, favorite
  event ids, onboarding progress.
- **Needs `flutter_secure_storage`** (Keychain / Keystore-backed): auth tokens you manage
  yourself, refresh tokens, anything you'd call a credential, PII you cache.
- **Should not be on device at all:** other users' PII, precise coordinates history, moderation
  decisions.

Note: the Firebase Auth SDK manages its own token persistence — don't report that as insecure
storage. The finding is *your* data, in *your* store.

Cache invalidation is a security concern here too: a multi-tier cache (memory → Hive/prefs →
Firestore) must be **cleared on logout**. Grep the logout path and diff it against every cache
writer. Leftover cached PII after logout on a shared device is a real M9 finding.

## 3. Logging (leaks at scale)

```bash
grep -rn "print(\|debugPrint(" lib/ --include="*.dart" | head -30
grep -rnE "(log|logger|Logger)[A-Za-z.]*\(.*(token|password|email|uid|lat|lng|phone)" lib/ --include="*.dart"
```
- Raw `print`/`debugPrint` in release builds writes to logcat/console where any app with log
  access (or a connected cable) can read it.
- A logger that ships to Crashlytics/analytics turns a local leak into a central PII store.
  Check that error objects and request bodies are not attached wholesale.
- Verify logging is compiled out or gated in release (`kDebugMode`).

## 4. Build hardening (M7 / MASVS-RESILIENCE)

```bash
grep -rn "obfuscate\|split-debug-info" android/ ios/ *.md 2>/dev/null | head
grep -rn "minifyEnabled\|shrinkResources\|proguard" android/app/build.gradle*
```
Recommend (as Hardening, not as vulnerabilities):
```bash
flutter build appbundle --release --obfuscate --split-debug-info=build/symbols
```
Keep the symbol directory — without it, crash reports are unreadable. Obfuscation and native
tricks slow attackers; they never replace backend authorization. Say that explicitly in the
report so nobody treats it as a fix for an authz finding.

Also check: `android:debuggable` not set in release, `allowBackup` posture, no debug signing
config in release, ProGuard/R8 rules don't accidentally strip security code.

## 5. App Check init (M8)

```bash
grep -rn "FirebaseAppCheck\|AndroidProvider\|AppleProvider" lib/ --include="*.dart"
```
Correct pattern:
```dart
if (!kIsWeb) {
  await FirebaseAppCheck.instance.activate(
    androidProvider: kDebugMode ? AndroidProvider.debug : AndroidProvider.playIntegrity,
    appleProvider:   kDebugMode ? AppleProvider.debug  : AppleProvider.deviceCheck,
  );
}
```
Audit points:
- A debug provider reachable in a release build = App Check bypass for anyone with the debug
  token. Confirm the `kDebugMode` branch, and that debug tokens aren't committed.
- **A platform skipped at init (e.g. `!kIsWeb`) will be rejected once enforcement is on.**
  That is a launch-blocking interaction between client init and server enforcement — report it
  when reviewing either side.
- App Check must be activated **before** the first Firebase call that needs a token.

## 6. Transport (M5)

```bash
grep -rn "http://" lib/ --include="*.dart" | grep -v localhost | grep -v 127.0.0.1
grep -rn "badCertificateCallback\|allowBadCertificates\|HttpOverrides" lib/ --include="*.dart"
```
- Firebase SDK traffic is TLS by default — do not report "HTTPS not enforced" generically.
- A `badCertificateCallback` returning true, or a global `HttpOverrides` that disables
  validation, is a **real** High finding: it defeats TLS for every request.
- Certificate pinning is appropriate for high-sensitivity apps; treat its absence as Hardening
  unless the app moves money or health data. Note the operational cost: pinning breaks on cert
  rotation and needs a release to fix.

## 7. Platform surface (MASVS-PLATFORM)

- **Deep links / app links:** every route reachable from a URL must re-check authorization
  server-side. A deep link that opens an event detail screen must not assume the user was
  allowed to navigate there.
- **Exported Android components:** check `AndroidManifest.xml` for `android:exported="true"` on
  activities/receivers that shouldn't be.
- **Screenshot / recents protection** for screens showing PII (`FLAG_SECURE`) — Hardening.
- **Clipboard:** don't copy tokens or PII silently.
- **WebViews:** if any exist, check JS bridges and that untrusted HTML never renders (that's
  where CWE-79 XSS re-enters a mobile app).
- **Permissions:** location, camera, contacts — request the minimum, at point of use. Over-broad
  permissions are a privacy finding (M6) and a Play review risk.

## 8. Client-side "security" that isn't

Flag these as **Insecure Design (A06)** whenever the client is the only enforcement point:
- Hiding an admin button instead of gating the endpoint
- Validating input only in the widget
- Age checks done in the UI while rules accept any `minAge`
- Rate limiting with a local timer/counter
- Trusting a locally cached `isAdmin` / `accountStatus`

The test is always: *if the attacker skips the UI entirely and calls the backend directly, what
stops them?* If the answer is "nothing", the control does not exist.
</content>
