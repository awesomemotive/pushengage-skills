# Cross-platform PushEngage Best Practices (XP-BP)

Rules in this file apply regardless of platform. Each rule has a stable ID (`XP-BP-NN`) so audit findings, debugging output, and best-practice nudges can cite a specific rule without restating it.

Platform-specific rules live in each platform spoke's `best-practices.md` (e.g., `IOS-BP-NN`, `ANDROID-BP-NN`).

---

## XP-BP-01 — Never hardcode the App ID in source

**Rule:** Don't embed the PushEngage App ID directly in code that ships to source control.

**Why:** Mostly hygiene (it's not as sensitive as a server-side API key), but treating it as configuration prevents accidental cross-environment leaks (e.g., a production App ID slipping into a debug build) and makes per-environment override easier.

**How to apply:**
- Read it from a config file, environment-flag-aware build setting, or platform-specific equivalent.
- Audit check: flag any literal UUID string in `setAppId(...)` / `setAppID(id:)` (iOS) / equivalent that matches the App ID UUID format `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.

## XP-BP-02 — Use a separate PushEngage site for testing vs production

**Rule:** Don't send test notifications through your production site.

**Why:** Test sends pollute production analytics, can accidentally reach real subscribers, and make it harder to debug a real failure when you can't tell test traffic from prod.

**How to apply:** Create a second PushEngage site, configure your debug/test build to use its App ID, swap at build time.

## XP-BP-03 — Request notification permission at a meaningful moment, not on app launch

**Rule:** Don't call `requestNotificationPermission` (or platform equivalent) in your app's first-launch code path. Wait until the user has seen a relevant screen and there's clear value in subscribing.

**Why:** Apple HIG and Android Material guidance both say this; users who deny on the cold-launch prompt have very few paths to re-grant. The PushEngage SDK auto-subscribes when permission is granted, so denial = lost subscriber, often permanently.

**How to apply:** Defer the permission call behind a user action (e.g., "Get price-drop alerts" tap). Audit check: if `requestNotificationPermission` is called inside `application(_:didFinishLaunchingWithOptions:)` (iOS) or `onCreate` (Android), flag it.

## XP-BP-04 — Use stable, low-cardinality tags

**Rule:** Don't dump everything into subscriber attributes / tags. Keep them stable and bounded.

**Why:** PushEngage's targeting performance and dashboard usability degrade as the cardinality of attribute values explodes. A `last_seen_iso8601` attribute with millions of unique values is a footgun; `engagement_tier=high|medium|low` is useful.

**How to apply:** Prefer enumerated values. Don't store free-form text, timestamps, or per-request data as subscriber attributes — use `sendGoal` events for those.

## XP-BP-05 — Treat subscriber hash and device token as PII

**Rule:** Don't log raw subscriber hash, device token, or `getSubscriberId()` results to crash reports, analytics, or anywhere outside the PushEngage SDK's own storage.

**Why:** These are user identifiers tied to a device. Privacy reviews and App Store / Play Store compliance both care.

**How to apply:** If a customer's debug code prints the result of `getSubscriberId`, suggest replacing it with a redacted form.

## XP-BP-06 — Pin the SDK version per the platform spoke's version matrix

**Rule:** Native iOS and Android integrations pin to an exact version (`pod 'PushEngage', '1.0.0'`, exact gradle coordinate). Flutter and React Native follow their own spoke rules (FLUTTER-BP-10, RN-BP-10), which allow a caret pin at the plugin layer.

**Why:** The native SDKs are now at iOS 1.0.0 / Android 0.1.0 and the plugins at Flutter 1.0.0 / React Native 1.0.1; a non-exact range on a native pod or gradle dep can still quietly upgrade and break a working integration, so pin native deps exactly. The plugin layers (pub/npm) pin their transitive native SDKs exactly, so the caret there doesn't reintroduce drift.

**How to apply:** Audit check: flag any non-exact version pin in Podfile / Package.swift / build.gradle for the PushEngage SDK. For pubspec.yaml / package.json, defer to FLUTTER-BP-10 / RN-BP-10 — don't flag the caret form those rules prescribe.

## XP-BP-07 — Disable verbose logs in production builds

**Rule:** `enableLogging` (or equivalent) must be `false` / off in release builds.

**Why:** SDK logs include subscriber/device identifiers and full notification payloads. Shipping them in a production build exposes user data and bloats device logs.

**How to apply:** Audit check: flag any unconditional `enableLogging = true` / `enableLogs(true)` that isn't gated by `#if DEBUG` (iOS), `BuildConfig.DEBUG` (Android), or equivalent.

---

Spokes citing these rules: just write the rule ID (e.g., "violates XP-BP-03") with a one-line summary; don't restate the whole rule. The customer reading the finding can click into this file if they want the why.
