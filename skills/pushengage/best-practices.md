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

**Rule:** Native iOS and Android integrations pin to an exact version (`pod 'PushEngage', '1.1.0'`, exact gradle coordinate). Flutter and React Native follow their own spoke rules (FLUTTER-BP-10, RN-BP-10), which allow a caret pin at the plugin layer.

**Why:** The native SDKs are now at iOS 1.1.0 / Android 1.0.0 and the plugins at Flutter 1.1.0 / React Native 1.1.0 (the first versions with In-App Messaging); a non-exact range on a native pod or gradle dep can still quietly upgrade and break a working integration, so pin native deps exactly. The plugin layers (pub/npm) pin their transitive native SDKs exactly, so the caret there doesn't reintroduce drift.

**How to apply:** Audit check: flag any non-exact version pin in Podfile / Package.swift / build.gradle for the PushEngage SDK. For pubspec.yaml / package.json, defer to FLUTTER-BP-10 / RN-BP-10 — don't flag the caret form those rules prescribe.

## XP-BP-07 — Disable verbose logs in production builds

**Rule:** `enableLogging` (or equivalent) must be `false` / off in release builds.

**Why:** SDK logs include subscriber/device identifiers and full notification payloads. Shipping them in a production build exposes user data and bloats device logs.

**How to apply:** Audit check: flag any unconditional `enableLogging = true` / `enableLogs(true)` that isn't gated by `#if DEBUG` (iOS), `BuildConfig.DEBUG` (Android), or equivalent.

## XP-BP-08 — IAM trigger event names and condition types must match the dashboard exactly

**Rule:** The `eventName` passed to `triggerIAMEvent(...)` must equal the campaign's trigger event **character-for-character, case included**. Pass parameter values in their natural type (numbers as numbers) and have the dashboard condition typed `number` / `version` / `boolean` when the comparison isn't textual.

**Why:** Matching is exact — `Cart_Abandoned` never matches `cart_abandoned`, and there is no fuzzy fallback. Values are stringified before matching; the condition's declared type decides how that text is compared, so a `string`-typed `greater than 100` compares alphabetically (or never matches) instead of numerically. A blank name is rejected with `400 Event name is required` rather than silently matching nothing.

**How to apply:** When writing or reviewing a `triggerIAMEvent` call, ask the customer for the dashboard's exact event string and compare. If a campaign "never shows", this is checklist item 4 in `in-app-messaging.md`. No static audit — the dashboard side isn't in the repo.

## XP-BP-09 — Route IAM in-app navigation through `custom` actions; `open_url` is external http(s) only

**Rule:** Don't configure IAM `open_url` buttons with app-scheme deep links (`myapp://…`) — they are refused. For in-app navigation use a `custom` action, give it an Action name on the dashboard, and switch on `parameters["action"]` in the platform's custom-action hook (never on `actionId`, which is a dashboard-generated internal key). Don't expect IAM taps to arrive on the push deep-link listeners (`setNotificationOpenHandler`, `deepLinkStream`, `onValueChanged`).

**Why:** `open_url` opens the external browser and accepts only `http`/`https` (a missing scheme becomes `https`). A `url` parameter on a `custom` action is forwarded verbatim, never opened. IAM and push are separate pipelines on every platform.

**How to apply:** If the customer says "the in-app button with my deep link does nothing", check the action type; if it's `open_url` with a custom scheme, switch it to `custom` and add the handler. If they wired IAM routing into their push listener, move it to the IAM hook.

## XP-BP-10 — IAM needs only the App ID, but subscriber-data audiences need a push subscription

**Rule:** Never make `requestNotificationPermission()` / `subscribe()` a prerequisite for in-app messaging. Do warn that campaigns whose audience uses subscriber data (`segments`, `attr.*`, `city`/`state`/`geo_country`, `has_unsubscribed`) are held back until the device is a push subscriber and the subscriber record has been fetched once.

**Why:** IAM syncs and displays on the App ID alone. Subscriber-backed audience rules are deliberately *skipped* (not evaluated) without a subscriber record, so an IAM-only integration never shows those campaigns, and a fresh install typically first evaluates them on the second app open. Device-data rules are unaffected.

**How to apply:** When a customer plans IAM without push, or tests on a simulator / build without push entitlements, tell them segment- and attribute-targeted campaigns won't appear and suggest device-data targeting for those tests.

## XP-BP-11 — Every IAM campaign must carry its own way out

**Rule:** Each in-app campaign should contain a button wired to a **Dismiss** action (or set an auto-dismiss duration). Banners and full-screen messages especially, since tap-outside has little or no outside area there.

**Why:** The SDK draws **no** close button. A banner with no auto-dismiss, no tap-to-dismiss and no working button is genuinely unclosable. Any recognised button dismisses the message, so one Dismiss button is enough.

**How to apply:** Dashboard-side authoring rule — surface it when walking the customer through campaign creation or when they report a message they "can't close". No static audit.

---

Spokes citing these rules: just write the rule ID (e.g., "violates XP-BP-03") with a one-line summary; don't restate the whole rule. The customer reading the finding can click into this file if they want the why.
