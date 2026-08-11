# Flutter Audit Checks (FLUTTER-AUDIT)

Static audit rules for Flutter integrations. Each check runs against specific files and surfaces findings categorized **Likely cause** / **Worth fixing** / **Cosmetic**. Cite the rule ID alongside related `FLUTTER-BP-NN` (best-practice) and `KI-FNNN` (known-issue) IDs.

Run order: read all relevant files once, apply each check, group findings by category before reporting.

---

## FLUTTER-AUDIT-01 — Stale `pushengage_flutter_sdk` pin (pre-`1.0.0`)

**Files to check:** `pubspec.yaml`.

**Pattern:** `pushengage_flutter_sdk: ^0.0.1` (or any value `< 1.0.0`, including `^0.0.2`, `>= 0.0.1`, `any`, or a major-only constraint).

**Categorize:** Likely cause (if customer reports runtime "no such method" errors on modern API calls).

**References:** FLUTTER-BP-10, KI-F001.

**Fix:** bump to `^1.0.0` in `pubspec.yaml`, then `flutter pub upgrade pushengage_flutter_sdk`. The current published version is `1.0.0`; older pins like `^0.0.1` are missing most of the modern API. Quote `install.md` Step 1.

---

## FLUTTER-AUDIT-02 — `setAppId` called from a widget's `initState` instead of `main()`

**Files to check:** `lib/main.dart`, plus any file containing `PushEngage.setAppId(` (grep across `lib/`).

**Pattern:** A call to `PushEngage.setAppId(...)` that appears (a) inside a `State<...>` class's `initState()`, OR (b) after `runApp(...)` in `main()`, OR (c) inside any widget's `build`, `didChangeDependencies`, etc.

**Categorize:** Worth fixing.

**References:** FLUTTER-BP-01.

**Fix:** move `setAppId` into `main()` AFTER `WidgetsFlutterBinding.ensureInitialized()` and BEFORE `runApp(...)`. Quote `init-code.md` Section 1.

---

## FLUTTER-AUDIT-03 — Stale / wrong NSE / NCE Podfile pod pin

**Files to check:** `ios/Podfile`.

**Pattern:** Inside an NSE or NCE target block, either (a) `pod 'PushEngage', ...` (ANY version — the app pod no longer belongs in an extension block under the 1.0.0 split), OR (b) `pod 'PushEngageExtension'` with a version `< 1.0.0`.

**Categorize:** Likely cause (if customer reports rich-push image not loading or NSE-related build issues).

**References:** FLUTTER-BP-06, KI-F011.

**Fix:** change to `pod 'PushEngageExtension', '1.0.0'` in the NSE and NCE target blocks (the extension-safe pod that `PushEngage` depends on). After editing, run `cd ios && pod install` then clean build (Xcode → Product → Clean Build Folder).

---

## FLUTTER-AUDIT-04 — App pod `PushEngage` pinned inside an NSE / NCE target block

**Files to check:** `ios/Podfile`.

**Pattern:** Inside an NSE or NCE target block (e.g., `target 'NotificationServiceExtension' do ... end` / `target 'NotificationContentExtension' do ... end`), a line `pod 'PushEngage', ...` appears. Under iOS SDK 1.0.0 the extension targets must link the extension-safe `PushEngageExtension` pod, NOT the app pod.

> This check was inverted in `1.0.0`. The old `APPLICATION_EXTENSION_API_ONLY = 'No'` `post_install` requirement is obsolete — the SDK removed the conditional-compilation mechanism that needed it — so the absence of that override is no longer a finding. The real failure mode now is pinning the wrong pod in the extension block.

**Categorize:** Likely cause (if customer reports iOS build error "Application extensions cannot use PushEngage" or similar).

**References:** FLUTTER-BP-06, FLUTTER-AUDIT-03.

**Fix:** change the extension block's pin to `pod 'PushEngageExtension', '1.0.0'`, then `cd ios && pod install`. See `nse-setup.md` / `nce-setup.md`.

---

## FLUTTER-AUDIT-05 — `AndroidManifest.xml` missing explicit `POST_NOTIFICATIONS`

**Files to check:** `android/app/src/main/AndroidManifest.xml`.

**Pattern:** No `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` declaration above `<application>`.

**Categorize:** Worth fixing (clarity). The transitive native PushEngage Android SDK declares this permission in its own library manifest, and manifest merger propagates it — so the dialog normally appears even without an app-level entry. Escalate to Likely cause ONLY if the permission is also absent from the merged manifest (`android/app/build/intermediates/merged_manifests/...`) or a `tools:node="remove"` strips it.

**References:** KI-F004 (mirror of native KI-A001).

**Fix:** add `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` above the `<application>` element as an explicit declaration. Harmless on pre-13 Android.

---

## FLUTTER-AUDIT-06 — `google-services.json` at wrong path

**Files to check:** project tree.

**Pattern:** `google-services.json` exists at project root OR at `android/google-services.json`, but NOT at `android/app/google-services.json`.

**Categorize:** Likely cause (if Gradle build fails with `File google-services.json is missing. The Google Services Plugin cannot function without it.`).

**References:** KI-A006 (native Android spoke).

**Fix:** `mv google-services.json android/app/google-services.json` (or whichever wrong-path location). Re-sync Gradle from Android Studio or `./gradlew :app:assembleDebug`.

---

## FLUTTER-AUDIT-07 — Single hardcoded App ID UUID on cross-platform Flutter app

**Files to check:** `lib/main.dart`, plus any file containing `PushEngage.setAppId(`.

**Pattern:** `setAppId('<uuid-literal>')` (string literal, not a `Platform.isIOS ? ... : ...` ternary), AND both `ios/` and `android/` folders exist at the project root.

**Categorize:** Likely cause (if customer reports "works on one platform but not the other").

**References:** FLUTTER-BP-03, KI-F006.

**Fix:** wrap the hardcoded UUID with `Platform.isIOS ? '<IOS_UUID>' : '<ANDROID_UUID>'`. Each PushEngage site generates its own App ID UUID; reusing one across platforms is unsupported. If the customer is shipping to one platform only, this rule does not apply.

---

## FLUTTER-AUDIT-08 — `AppDelegate.swift` missing `PushEngage.swizzleInjection(isEnabled: true)`

**Files to check:** `ios/Runner/AppDelegate.swift`.

**Pattern:** The file does NOT contain `PushEngage.swizzleInjection(isEnabled: true)` inside `override init()` (or anywhere). And it does NOT contain the four manual-mode forwards (which would indicate intentional manual integration).

**Categorize:** Likely cause (if customer reports iOS pushes not arriving despite Dart `setAppId` succeeding).

**References:** (mirrors `IOS-BP-02` and `IOS-AUDIT-04` from the iOS-native spoke for the alternative manual case).

**Fix:** add the canonical AppDelegate from `init-code.md` Section 2. Default to swizzling mode unless the customer has explicitly opted into manual mode (rare; only needed if another iOS SDK also swizzles).

---

## FLUTTER-AUDIT-09 — `PushEngage_App_Group_Key` missing or mismatched

**Files to check:** `ios/Runner/Info.plist`, NSE target's `Info.plist` (typically `ios/NotificationServiceExtension/Info.plist`).

**Pattern:** Either:
- `Info.plist` of the main `Runner` target is missing the `PushEngage_App_Group_Key` key, OR
- the NSE's `Info.plist` is missing it, OR
- both have the key but the values differ.

**Categorize:** Likely cause (if customer reports rich-push image not loading OR delivery analytics broken).

**References:** KI-F007.

**Fix:** set `PushEngage_App_Group_Key` to the same string (e.g., `group.<your-bundle>.pushengage`) in BOTH plists. Verify the App Groups capability is enabled on both targets (Xcode → target → Signing & Capabilities → App Groups), and the same group ID is checked on both. Quote `install.md` Step 3d and `nse-setup.md` step 12.

---

## FLUTTER-AUDIT-10 — `pod 'PushEngage'` declared in main `Runner` target

**Files to check:** `ios/Podfile`.

**Pattern:** Inside `target 'Runner' do ... end`, a line `pod 'PushEngage', <version>` that pins a version OTHER than `1.0.0` (the version the Flutter plugin pulls transitively). An explicit `pod 'PushEngage', '1.0.0'` matching the transitive pin is NOT a finding — the SDK's own example Podfile does exactly this and builds fine.

**Categorize:** Cosmetic if pinned to `1.0.0` (redundant only); Worth fixing if it pins a different version (risks a version mismatch with the transitive pin).

**References:** FLUTTER-BP-06.

**Fix:** either drop the redundant line from the `Runner` target, or align it to `1.0.0`. NSE and NCE targets DO need an explicit pin, but for the extension-safe pod (`pod 'PushEngageExtension', '1.0.0'`), since they're separate from Flutter's plugin resolution.

---

## FLUTTER-AUDIT-11 — Unconditional `enableLogging(true)`

**Files to check:** all `lib/**/*.dart`.

**Pattern:** A call to `PushEngage.enableLogging(true)` that is NOT wrapped in `if (kDebugMode) { ... }` (or equivalent `if (!kReleaseMode)` / `assert(...)` guard).

**Categorize:** Cosmetic.

**References:** FLUTTER-BP-05.

**Fix:** wrap in `if (kDebugMode) { PushEngage.enableLogging(true); }`. Import `package:flutter/foundation.dart`. Not a build/integration breaker but a privacy concern in release builds (logs may include subscriber IDs and FCM/APNs tokens).

---

## FLUTTER-AUDIT-12 — `deepLinkStream.listen` registered post-root

**Files to check:** all `lib/**/*.dart`.

**Pattern:** A call to `PushEngage.deepLinkStream.listen(...)` that appears in any widget OTHER than the immediate child of `runApp(...)`. Common bad locations: inside a `FutureBuilder`, inside a route's `initState` (e.g., `HomePage`'s `initState` when `HomePage` is pushed after auth), inside `LaunchedEffect`-equivalent post-build hooks.

**Categorize:** Worth fixing (buffered taps can be dropped — the native drain fires on first stream access, and a broadcast stream with no listener discards events; Android cold-boot taps arrive on this stream, so late registration risks losing them).

**References:** FLUTTER-BP-02, KI-F003.

**Fix:** move the `.listen(...)` call to the root widget's `initState()`. Quote `init-code.md` Section 1 (deep-link listener block).

---

## FLUTTER-AUDIT-13 — Hardcoded App ID UUID in source

**Files to check:** `lib/main.dart` (and anywhere `setAppId` appears).

**Pattern:** `setAppId(...)` argument is a UUID string literal (not loaded from `String.fromEnvironment`, a config file, or `BuildConfig`-equivalent).

**Categorize:** Cosmetic.

**References:** XP-BP-01 (cross-platform — secrets / identifiers belong outside source).

**Fix:** move the UUID(s) to `String.fromEnvironment('PUSHENGAGE_APP_ID_IOS')` / `--dart-define` arguments at build time, or to a separate untracked config file. The App ID UUID is technically not a secret (it's transmitted in every API call as a public identifier), but the same XP-BP-01 hygiene applies for environment management (dev / test / prod).

---

## Categorization summary

| Category | When to use |
|---|---|
| **Likely cause** | The finding explains a symptom the customer is reporting RIGHT NOW. High signal: usually one finding total. |
| **Worth fixing** | Will cause a problem under predictable conditions (e.g., cold-boot tap lost, denied user can't recover). Not necessarily the current bug. |
| **Cosmetic** | No functional impact under current usage; worth addressing before release / scale-up. |

When reporting findings, present **Likely cause** first, then **Worth fixing**, then **Cosmetic** — and explicitly note which checks PASSED. The customer needs to know what was checked and ruled out, not just what failed.
