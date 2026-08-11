# Flutter PushEngage Best Practices (FLUTTER-BP)

Flutter-specific rules. Cite by rule ID in audit findings and debugging output. Cross-platform rules live in `../pushengage/best-practices.md` (`XP-BP-NN`).

---

## FLUTTER-BP-01 — Initialize `setAppId` in `main()` before `runApp`

**Rule:** Prefer calling `PushEngage.setAppId(...)` from `main()` AFTER `WidgetsFlutterBinding.ensureInitialized()` and BEFORE `runApp(...)`. Calling it later (e.g. from a widget's `initState()`) is acceptable when the App ID is only known at runtime.

**Why:** Subscriber registration begins as soon as `setAppId` resolves on the native side, so setting it early minimizes registration latency. This does NOT govern cold-boot deep-link delivery — a tap that launches the app is recovered via `getInitialNotification()` and the timing of `deepLinkStream.listen` (FLUTTER-BP-02), independent of `setAppId`. The SDK's own example app calls `setAppId` from `initState` deliberately, because its App ID is entered by the user at runtime — that is a valid pattern, not a bug. For a greenfield app with a known App ID, `main()` is the cleaner home.

**How to apply:** Audit check FLUTTER-AUDIT-02. Grep `lib/main.dart` for the position of the `setAppId` call relative to `runApp`. If `setAppId` appears inside a `State<...>` class or after `runApp`, flag with this rule ID. Quote the fix from `init-code.md`.

## FLUTTER-BP-02 — Register `deepLinkStream.listen` in the root widget's `initState`

**Rule:** The `PushEngage.deepLinkStream.listen(...)` call must run inside the root widget's `initState()`, BEFORE any async startup work (auth check, splash, remote config, A/B fetch, deferred deep-link route resolution). Cancel the subscription in `dispose()`.

**Why:** `deepLinkStream` is a broadcast stream, lazy-initialized on first access — that first access also signals native that Dart is ready, which drains the native event buffer into the stream. Events delivered to a broadcast stream with no listener are dropped, so the `.listen(...)` must happen at that first access (getter + listen together, as in `init-code.md`). Registering late also risks the capped native buffer dropping the oldest events (32, both platforms). On iOS, a cold-boot (force-quit) tap never reaches this stream — it's recovered via `getInitialNotification()`. Mirror of RN-BP-02.

**How to apply:** Audit check FLUTTER-AUDIT-12. Grep all Dart files for `deepLinkStream.listen(`. If the call appears in any widget that isn't the app root (the widget passed to `runApp`), flag with this rule ID and KI-F003. Also flag if the call is wrapped behind a `FutureBuilder`, `if (loaded) ...`, or post-auth route.

## FLUTTER-BP-03 — Use `Platform.isIOS ? IOS_APP_ID : ANDROID_APP_ID` for cross-platform builds

**Rule:** If the customer ships to BOTH platforms (Flutter project has both `ios/` and `android/` folders), the App ID passed to `setAppId` must be platform-conditional. Don't hardcode a single UUID.

**Why:** Each PushEngage site (Android, iOS) generates its own App ID UUID. Re-using one UUID across platforms results in one platform's subscribers being silently misrouted or never registered. It's the most common multi-platform Flutter integration mistake. (KI-F006.)

**How to apply:** Audit check FLUTTER-AUDIT-07. If `lib/main.dart` calls `setAppId` with a string literal AND both `ios/` and `android/` exist in the project root, flag with this rule ID. Propose the `Platform.isIOS ? IOS_APP_ID : ANDROID_APP_ID` pattern. If the customer only ships to one platform, this rule does not apply.

## FLUTTER-BP-04 — Defer `requestNotificationPermission()` to a meaningful moment

**Rule:** Do NOT call `PushEngage.requestNotificationPermission()` from `main()`, app-root `initState`, or first launch. Trigger it on a user action that makes the prompt make sense — after onboarding, when the user taps "Enable notifications," or after they complete a key first-use step.

**Why:** Both Android (API 33+) and iOS only surface the system permission dialog ONCE in most cases. If the user denies the prompt during cold boot (before they've seen the app's value), the only path back is the system settings app. This is the Flutter analog of XP-BP-03 and ANDROID-BP-03.

**How to apply:** Recommend during `permission-flow.md`. No automatic audit (the absence of a call in `main()` is the default state). If you see `requestNotificationPermission()` in `main()` or in the root widget's `initState`, raise it as a "worth fixing" finding.

## FLUTTER-BP-05 — Gate `enableLogging(true)` behind `kDebugMode`

**Rule:** Wrap `PushEngage.enableLogging(true)` in `if (kDebugMode) { ... }` (import `package:flutter/foundation.dart`). Never ship release builds with logging on.

**Why:** SDK logs may include subscriber IDs, FCM/APNs tokens, deep-link payloads, and request/response bodies. Release builds emitting these are a privacy and security smell. Flutter analog of XP-BP-07 and ANDROID-BP-06.

**How to apply:** Audit check FLUTTER-AUDIT-11. Grep all Dart files for `PushEngage.enableLogging(true)`. If found outside a `kDebugMode` (or equivalent debug-mode gate), flag with this rule ID. Cosmetic category — not a build/integration breaker, but worth fixing before release.

## FLUTTER-BP-06 — Adding `pod 'PushEngage'` to the main `Runner` target is redundant

**Rule:** The Flutter plugin's podspec already pins `pod 'PushEngage', '1.0.0'` as a transitive dependency, so you do NOT need to add it to the `Runner` target. If it is present and pinned to `1.0.0` it is harmless (the SDK's own example Podfile does this). Only add `pod 'PushEngageExtension', '1.0.0'` to the NSE and NCE target blocks (which don't go through Flutter's plugin resolution and must link the extension-safe pod, NOT the app pod).

**Why:** A redundant `pod 'PushEngage', '1.0.0'` in `Runner` resolves to the same version the plugin expects and builds fine. The risk only appears if it pins a DIFFERENT version — then the Podfile resolver picks the highest constraint and can quietly bump the native SDK behind the plugin's back, leading to symbol conflicts.

**How to apply:** Audit check FLUTTER-AUDIT-10. Read `ios/Podfile`. If `target 'Runner' do ... pod 'PushEngage', <version> ... end` pins a version other than `1.0.0`, flag it (worth fixing). A matching `1.0.0` is cosmetic/redundant only.

## FLUTTER-BP-07 — Treat subscriber IDs and profile IDs as PII

**Rule:** When using `getSubscriberId()`, `addProfileId(...)`, or `getSubscriberDetails(...)`, treat the values as personal identifiers. Don't log them outside debug builds; don't ship them to third-party analytics platforms without disclosure; don't put them in URLs or query strings; don't write them to shared preferences without considering backup exclusion.

**Why:** `subscriber_hash` is a per-device personal identifier. `profile_id` is typically a username or email hash that links the device to a real-world identity. Mirror of XP-BP-05.

**How to apply:** Recommend during integration wrap-up if these methods are present in the customer's code. No automatic audit (legitimate uses are common; the concern is downstream handling).

## FLUTTER-BP-08 — Cancel `deepLinkStream` subscriptions in `dispose()`

**Rule:** When `.listen(...)` on `deepLinkStream` returns a `StreamSubscription`, store it in a class field and call `.cancel()` in the widget's `dispose()`.

**Why:** `deepLinkStream` is a broadcast stream — multiple listeners accumulate. In production this is mostly cosmetic (subscriptions stay alive as long as their widgets do), but during development hot-reload re-creates widgets while old subscriptions linger, fanning out duplicate deep-link callbacks. Best practice across both production and DX.

**How to apply:** Cosmetic. Recommend during `init-code.md` walkthrough. No automatic audit unless `deepLinkStream.listen` is called without a stored `StreamSubscription` variable.

## FLUTTER-BP-09 — Run `cd ios && pod install` after every `flutter pub get`

**Rule:** Every time `pubspec.yaml`'s `pushengage_flutter_sdk` entry changes (added, version bumped) and the customer runs `flutter pub get`, they must follow it with `cd ios && pod install` before the iOS build will resolve correctly.

**Why:** Flutter's pub resolver does NOT auto-trigger CocoaPods resolution. The Flutter plugin's podspec exists in the plugin's local `.symlinks/plugins/pushengage_flutter_sdk/ios/` after `pub get`, but the customer's Podfile doesn't see it until `pod install` runs. Skipping this leads to "Pod not found" or "PushEngage pod missing" errors at the next Xcode build. (KI-F005.)

**How to apply:** Surface as a checklist item during `install.md` Step 3. Audit equivalent: if `ios/Podfile.lock` doesn't mention `pushengage_flutter_sdk` but `pubspec.yaml` does, flag as a missing `pod install` step.

## FLUTTER-BP-10 — Pin `pushengage_flutter_sdk` with `^1.0.0`, not a wide range

**Rule:** Use a caret-pinned version in `pubspec.yaml`: `pushengage_flutter_sdk: ^1.0.0`. Don't use `>=1.0.0` or `any`.

**Why:** As of `1.0.0` the SDK is on standard semver, so `^1.0.0` correctly allows non-breaking `1.x` upgrades while excluding a future `2.0.0` that could change the API. The old pre-1.0 caveat (any increment could break) no longer applies — this is now plain caret semantics. Using `any` or an open `>=` still risks `pub upgrade` quietly pulling a major bump with a different API shape, so keep the caret. Mirror of XP-BP-06 and the iOS spoke's exact-pin rule.

**How to apply:** Audit check FLUTTER-AUDIT-01 (combined with KI-F001 stale-version detection). If `pubspec.yaml` shows `^0.0.1`, any pre-`1.0.0` pin, `>= 1.0.0`, `any`, or a major-version-only constraint, flag with this rule ID. Quote the fix.

---

Spokes citing Flutter rules: just write the rule ID (e.g., "violates FLUTTER-BP-02, deepLinkStream listener registered post-auth"). Customer can click into this file for the why.
