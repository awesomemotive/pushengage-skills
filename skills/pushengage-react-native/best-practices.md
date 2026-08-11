# React Native PushEngage Best Practices (RN-BP)

React Native-specific rules. Cite by rule ID in audit findings and debugging output. Cross-platform rules live in `../pushengage/best-practices.md` (`XP-BP-NN`).

---

## RN-BP-01 — Initialize `setAppId` in `index.js` before `AppRegistry.registerComponent`

**Rule:** Prefer calling `PushEngage.setAppId(...)` from `index.js` (or `index.tsx`) BEFORE the `AppRegistry.registerComponent(...)` call. Calling it later — from `App.tsx` or a component effect — is acceptable when the App ID is only known at runtime.

**Why:** Subscriber registration begins as soon as `setAppId` resolves on the native side, so early placement minimizes registration latency. It does NOT govern cold-boot deep-link delivery — a tap that launches the app is recovered via `getInitialNotification()` and the timing of `onValueChanged` (RN-BP-02), independent of `setAppId`. The SDK's own example app calls `setAppId` from a component rather than at module top-level, and it works — that placement is valid, not a bug. For a greenfield app with a known App ID, `index.js` top-level is simply the cleaner default.

**How to apply:** Audit check RN-AUDIT-02. Grep all `*.ts` / `*.tsx` / `*.js` files for `PushEngage.setAppId(`. If the call appears anywhere other than `index.js` / `index.tsx` at module-top level, flag with this rule ID.

## RN-BP-02 — Register `onValueChanged` in the root component's `useEffect`

**Rule:** The `PushEngage.onValueChanged(...)` subscription must be created inside the root component's `useEffect` (the component passed to `AppRegistry.registerComponent`), with an empty dependency array `[]`, BEFORE any async startup work (auth check, splash, deferred routing, navigation gating). Return a cleanup function that calls `subscription.remove()`.

**Why:** `onValueChanged` is a Turbo Module event emitter. The native side queues incoming deep links until the first subscriber attaches. If the customer registers the listener in a downstream screen's `useEffect` (post-auth, post-splash, behind a navigation gate), the native queue drains before the listener exists and the cold-boot tap is silently lost. Same root cause as Flutter KI-F003.

**How to apply:** Audit check RN-AUDIT-12. Grep all source files for `PushEngage.onValueChanged(`. If the call appears in any component other than the one passed to `AppRegistry.registerComponent` (typically `App.tsx` or the navigator root), flag with this rule ID and KI-RN003.

## RN-BP-03 — Use `Platform.OS === 'ios' ? IOS_APP_ID : ANDROID_APP_ID` for cross-platform builds

**Rule:** If the project ships to both stores (both `ios/` and `android/` folders exist and are actively used), `setAppId` must be platform-conditional. Don't hardcode a single UUID.

**Why:** Each PushEngage site (Android, iOS) issues its own App ID UUID. Reusing one UUID across platforms results in one platform's subscribers being silently misrouted or never registered. (KI-RN006.)

**How to apply:** Audit check RN-AUDIT-07. If `setAppId` is called with a string literal AND both `ios/` and `android/` exist in the project root, flag with this rule ID. Propose `Platform.OS === 'ios' ? ...` pattern. If the customer ships to one platform only, this rule does not apply.

## RN-BP-04 — Defer `requestNotificationPermission()` to a meaningful moment

**Rule:** Do NOT call `PushEngage.requestNotificationPermission()` from `index.js`, root `useEffect`, or first launch. Trigger it on a user action that makes the prompt make sense.

**Why:** Both Android (API 33+) and iOS only surface the system permission dialog ONCE in most cases. Denial during cold boot is effectively permanent. Cross-platform analog of XP-BP-03 and FLUTTER-BP-04.

**How to apply:** Recommend during `permission-flow.md`. Flag any `requestNotificationPermission` call in `index.js` or root-component `useEffect` as a "worth fixing" finding.

## RN-BP-05 — Gate `enableLogging(true)` behind `if (__DEV__)`

**Rule:** Wrap `PushEngage.enableLogging(true)` in `if (__DEV__) { ... }`. Never ship release builds with logging on.

**Why:** SDK logs may include subscriber IDs, FCM/APNs tokens, deep-link payloads, request/response bodies. Release builds leaking these are a privacy and security smell. Always gate `enableLogging(true)` behind `__DEV__`.

**How to apply:** Audit check RN-AUDIT-11. Grep all source files for `PushEngage.enableLogging(true)`. If not wrapped in a `__DEV__` (or equivalent) guard, flag. Cosmetic category.

## RN-BP-06 — Do NOT add `pod 'PushEngage'` to the main app's `Runner`-style target

**Rule:** The RN plugin pins `pod 'PushEngage', '1.0.0'` as a transitive dependency. Do NOT redeclare it in the customer's main app target in `ios/Podfile`. Only add `pod 'PushEngageExtension', '1.0.0'` (the extension-safe pod, new with the 1.0.0 two-pod split) to NSE/NCE target blocks (which don't go through RN's plugin resolution).

**Why:** Redeclaring the pod in the main target risks resolving a different version than what the RN plugin expects → symbol conflicts, runtime crashes, build-time errors when the iOS-native API surface diverges between versions.

**How to apply:** Audit check RN-AUDIT-10. Read `ios/Podfile`. If `target '<AppName>' do ... pod 'PushEngage' ... end` appears (main app target, not NSE/NCE), flag with this rule ID.

## RN-BP-07 — Treat subscriber IDs and profile IDs as PII

**Rule:** When using `getSubscriberId()`, `addProfileId(...)`, or `getSubscriberDetails(...)`, treat the values as personal identifiers. Don't log in production; don't ship to third-party analytics without disclosure; don't put in URLs.

**Why:** `subscriber_hash` is per-device personal identifier. `profile_id` is typically a username/email hash linking the device to a real-world identity. Cross-platform analog of XP-BP-05 and FLUTTER-BP-07.

**How to apply:** Recommend during integration wrap-up if these methods appear in the customer's code. No automatic audit.

## RN-BP-08 — Return `() => subscription.remove()` from `useEffect`

**Rule:** When subscribing to `PushEngage.onValueChanged(...)` (or any future event emitter), store the returned `EventSubscription` and return a cleanup function from the `useEffect` that calls `subscription.remove()`.

**Why:** Without cleanup, hot-reloads (development) and unmount cycles (production) leave dead listeners attached. The native side fan-outs deep-link events to every attached listener, so duplicate callbacks fire. In production this is mostly cosmetic; in dev it's painful.

**How to apply:** Recommend during `init-code.md` walkthrough. Audit hint: if `onValueChanged` appears inside a `useEffect` without a returned cleanup function, flag as a "worth fixing" finding.

## RN-BP-09 — Run `cd ios && pod install` after every `npm install` / `yarn add` touching the plugin

**Rule:** Every time `package.json`'s `@pushengage/pushengage-react-native` entry changes (added, version bumped) and the customer runs `npm install` or `yarn add`, they MUST follow it with `cd ios && pod install` before the iOS build will resolve correctly.

**Why:** npm/yarn don't auto-trigger CocoaPods resolution. After `npm install`, the plugin's symlinked podspec exists locally, but the customer's `Podfile.lock` doesn't know about it until `pod install` runs. Skipping = "Pod not found" / "PushEngage pod missing" errors. (KI-RN005.)

**How to apply:** Surface as a checklist item during `install.md`. Audit equivalent: if `ios/Podfile.lock` doesn't mention `pushengage-react-native` (or the PushEngage pod) but `package.json` does, flag as a missing `pod install` step.

## RN-BP-10 — Pin `@pushengage/pushengage-react-native` with a caret minor

**Rule:** Use a caret-pinned version in `package.json`: `"@pushengage/pushengage-react-native": "^1.0.0"`. Don't use `*`, `latest`, or `>=`.

**Why:** Caret-pinning lets patch upgrades through but not minor/major bumps that may include breaking changes. The team labels this release `1.0.0`; treat minor bumps as still potentially breaking until the API stabilizes. Mirror of XP-BP-06 / FLUTTER-BP-10.

**How to apply:** Audit check RN-AUDIT-13-adjacent. If `package.json` shows `"*"`, `"latest"`, `">=0.0.0"`, or any too-loose constraint, flag with this rule ID.

## RN-BP-11 — Apply `com.google.gms.google-services` plugin in `android/app/build.gradle`

**Rule:** `android/app/build.gradle` must contain `apply plugin: "com.google.gms.google-services"` (or the equivalent `id("com.google.gms.google-services")` if using plugins block). Without this, `google-services.json` is downloaded but never read at build time, and FCM token issuance silently fails.

**Why:** The Google Services Gradle plugin processes `google-services.json` and generates the resource entries the Firebase SDKs read. The buildscript classpath alone is not enough — the plugin must be **applied** in the app module. Make sure the customer's `android/app/build.gradle` applies it. (KI-RN010.)

**How to apply:** Audit check RN-AUDIT-15. Read `android/app/build.gradle` (or `.kts`). If `apply plugin: "com.google.gms.google-services"` (or `id("com.google.gms.google-services")`) is absent AND `android/app/google-services.json` exists, flag with this rule ID. Likely-cause category.

## RN-BP-13 — Recover cold-boot taps via `getInitialNotification` on iOS

**Rule:** In the root component's startup effect, call `getInitialNotification()` once (in addition to subscribing `onValueChanged`) and route the result if non-null.

**Why:** On iOS the force-quit launch tap is delivered before JS can attach to `onValueChanged`; call `getInitialNotification()` once on startup to recover it, otherwise that launch tap isn't seen by `onValueChanged`. Android resolves `null`, so the call is cross-platform-safe. The cold-boot tap is routed exclusively to the pull API and never replayed through `onValueChanged`. New in 1.0.0.

**How to apply:** Audit check RN-AUDIT-17. If `onValueChanged` is registered but `getInitialNotification` is never called, flag as "worth fixing" with KI-RN003.

---

Spokes citing RN rules: just write the rule ID (e.g., "violates RN-BP-02, onValueChanged listener in downstream screen"). Customer can click into this file for the why.
