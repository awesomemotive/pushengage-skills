# React Native Audit Checks (RN-AUDIT)

Static audit rules for React Native integrations. Each check runs against specific files and surfaces findings categorized **Likely cause** / **Worth fixing** / **Cosmetic**. Cite the rule ID alongside related `RN-BP-NN` (best-practice) and `KI-RN0NN` (known-issue) IDs.

Run order: read all relevant files once, apply each check, group findings by category before reporting.

---

## RN-AUDIT-01 — Stale import name `'pushengage-react-native'`

**Files to check:** all `*.js` / `*.ts` / `*.tsx` files in `src/` (or wherever the project's source lives — typically NOT in `node_modules` or `ios/`/`android/`).

**Pattern:** `import ... from 'pushengage-react-native'` (without the `@pushengage/` scope).

**Categorize:** Likely cause (Metro / TS errors: "Module 'pushengage-react-native' could not be resolved").

**References:** KI-RN001.

**Fix:** change to `import ... from '@pushengage/pushengage-react-native'`. The package is scoped — use the scoped name. Quote `install.md` Step 1.

---

## RN-AUDIT-02 — `setAppId` called outside `index.js` / `index.tsx`

**Files to check:** all `*.js` / `*.ts` / `*.tsx` (grep for `PushEngage.setAppId`).

**Pattern:** A call to `PushEngage.setAppId(...)` that appears anywhere OTHER than `index.js` or `index.tsx` at module top-level — e.g. inside `App.tsx`'s `useEffect`, a component's `componentDidMount`, or after `AppRegistry.registerComponent`.

**Categorize:** Cosmetic / optional. This is a preference, not a defect — the SDK's own example app places `setAppId` in a component and works. Only raise it when the customer wants to minimize subscriber-registration latency.

**References:** RN-BP-01.

**Fix (optional):** for the earliest possible registration, move `setAppId` into `index.js` BEFORE `AppRegistry.registerComponent`. Quote `init-code.md` Section 1. Do NOT present this as required.

---

## RN-AUDIT-03 — Stale / wrong NSE / NCE Podfile pin

**Files to check:** `ios/Podfile`.

**Pattern:** Inside an NSE or NCE target block, any `pod 'PushEngage', '<x>'` (the old app-side pod) — e.g. `~>0.0.5`, `0.0.5`, `0.0.6` — OR `pod 'PushEngageExtension', '<x>'` where `<x>` < `1.0.0`. As of 1.0.0 extension targets must link the extension-safe `PushEngageExtension` pod, not the app-side `PushEngage` pod.

**Categorize:** Likely cause — `pod 'PushEngage'` in an extension target (or a `PushEngageExtension` pin below 1.0.0) conflicts with the plugin's transitive exact `1.0.0` resolution and fails at `pod install` / breaks rich-push. A customer upgrading from the prior spoke baseline will have `pod 'PushEngage', '0.0.6'` in their NSE/NCE blocks — that is now both stale AND the wrong pod; it must move to `pod 'PushEngageExtension', '1.0.0'`.

**References:** RN-BP-06, KI-RN011.

**Fix:** change to `pod 'PushEngageExtension', '1.0.0'`. After editing, `cd ios && pod install`, then clean build (Xcode → Product → Clean Build Folder).

---

## RN-AUDIT-04 — `Podfile` extension targets / `post_install` override (1.0.0 two-pod split)

**Files to check:** `ios/Podfile`.

**Pattern:** As of 1.0.0 the old `APPLICATION_EXTENSION_API_ONLY = 'No'` / `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES = 'YES'` `post_install` override is **obsolete** — the conditional-compilation mechanism was replaced by the two-pod split (app target links `PushEngage`; extension targets link `PushEngageExtension`). Do NOT flag a *missing* override as a problem. Instead flag the real 1.0.0 misconfigurations:
- An NSE/NCE target that links `pod 'PushEngage'` instead of `pod 'PushEngageExtension', '1.0.0'` (covered by RN-AUDIT-03).
- A stale `APPLICATION_EXTENSION_API_ONLY = 'No'` override still present — harmless but obsolete; note it for cleanup, not as a build blocker.

**Categorize:** Cosmetic for a leftover stale override; defer the real extension-pod check to RN-AUDIT-03 (Likely cause).

**References:** RN-AUDIT-03.

**Fix:** ensure extension targets link `pod 'PushEngageExtension', '1.0.0'` (see `install.md` Step 3b). Remove any leftover `APPLICATION_EXTENSION_API_ONLY` / `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES` overrides when convenient — they are no longer needed in 1.0.0.

---

## RN-AUDIT-05 — `AndroidManifest.xml` missing explicit `POST_NOTIFICATIONS`

**Files to check:** `android/app/src/main/AndroidManifest.xml`.

**Pattern:** No `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` above `<application>`.

**Categorize:** Worth fixing (clarity). The transitive native PushEngage Android SDK declares this permission in its library manifest and manifest merger propagates it — so the dialog normally appears even without an app-level entry. Escalate to Likely cause ONLY if the permission is also absent from the merged manifest (`android/app/build/intermediates/merged_manifests/...`) or a `tools:node="remove"` strips it.

**References:** KI-RN004 (mirror of native KI-A001).

**Fix:** add the line as an explicit declaration; harmless on pre-13 Android. (This is also why **the SDK's example app**, whose Manifest omits the entry, still prompts — but don't copy the example verbatim.)

---

## RN-AUDIT-06 — `google-services.json` at wrong path

**Files to check:** project tree.

**Pattern:** `google-services.json` exists at project root OR at `android/google-services.json`, but NOT at `android/app/google-services.json`.

**Categorize:** Likely cause (if Gradle build fails with `File google-services.json is missing.`).

**References:** KI-A006 (native Android spoke).

**Fix:** `mv` the file to `android/app/google-services.json`. Re-sync Gradle.

---

## RN-AUDIT-07 — Single hardcoded App ID UUID on cross-platform RN app

**Files to check:** `index.js`, `index.tsx`, plus any file calling `PushEngage.setAppId`.

**Pattern:** `setAppId('<uuid-literal>')` (string literal, not a `Platform.OS === 'ios' ? ... : ...` ternary), AND both `ios/` and `android/` folders exist at the project root with active config.

**Categorize:** Likely cause (if customer reports "works on one platform but not the other").

**References:** RN-BP-03, KI-RN006.

**Fix:** wrap with `Platform.OS === 'ios' ? '<IOS_UUID>' : '<ANDROID_UUID>'`. Each PushEngage site issues its own UUID; reusing one across platforms is unsupported.

---

## RN-AUDIT-08 — AppDelegate missing PushEngage init

**Files to check:** whichever delegate file exists — `ios/<YourApp>/AppDelegate.swift` (RN ≥ 0.77 templates) or `ios/<YourApp>/AppDelegate.mm` (upgraded apps). Don't flag a "missing" `.mm` when the app has a Swift delegate.

**Pattern:** The delegate file does NOT enable swizzling in its initializer (`PushEngage.swizzleInjection(isEnabled: true)` / `[PushEngage swizzleInjectionWithIsEnabled:YES]`) OR does NOT call `setInitialInfo` (`PushEngage.setInitialInfo(for:with:)` / `[PushEngage setInitialInfoFor:application with:launchOptions]`) in `didFinishLaunchingWithOptions`.

**Categorize:** Likely cause (if iOS pushes don't arrive despite JS `setAppId` succeeding).

**References:** RN-BP-01.

**Fix:** apply the matching variant from `init-code.md` Section 2 (Swift deltas or ObjC++ template). Both init calls are required even in swizzling mode (RN apps still need explicit `setInitialInfo` for cold-boot replay).

---

## RN-AUDIT-09 — `PushEngage_App_Group_Key` missing or mismatched

**Files to check:** `ios/<YourApp>/Info.plist`, NSE target's `Info.plist` (typically `ios/NotificationServiceExtension/Info.plist`).

**Pattern:** Either:
- Main `Info.plist` missing the `PushEngage_App_Group_Key` key, OR
- NSE `Info.plist` missing it, OR
- Both present but with different values.

**Categorize:** Likely cause (if rich-push image not loading OR delivery analytics broken).

**References:** KI-RN007.

**Fix:** set `PushEngage_App_Group_Key` to the SAME string (e.g., `group.<your-bundle>.pushengage`) in BOTH plists. Verify App Groups capability is enabled on both Xcode targets with the same group ID checked.

---

## RN-AUDIT-10 — `pod 'PushEngage'` declared in main app target

**Files to check:** `ios/Podfile`.

**Pattern:** Inside `target '<YourAppName>' do ... end` (the main app target, NOT NSE/NCE), a line `pod 'PushEngage', ...` exists.

**Categorize:** Worth fixing.

**References:** RN-BP-06.

**Fix:** remove the line from the main target. The RN plugin pulls `PushEngage` transitively. NSE and NCE targets DO need explicit pins (they're separate from RN's plugin resolution).

---

## RN-AUDIT-11 — Unconditional `enableLogging(true)`

**Files to check:** all `*.js` / `*.ts` / `*.tsx`.

**Pattern:** A call to `PushEngage.enableLogging(true)` NOT wrapped in `if (__DEV__) { ... }` (or equivalent dev-mode guard).

**Categorize:** Cosmetic.

**References:** RN-BP-05.

**Fix:** wrap in `if (__DEV__) { PushEngage.enableLogging(true); }`. Not a build/integration breaker but a privacy concern in release (logs may include subscriber IDs and tokens). **The SDK's example app does NOT gate this — don't copy verbatim.**

---

## RN-AUDIT-12 — `onValueChanged` registered post-root

**Files to check:** all `*.js` / `*.ts` / `*.tsx`.

**Pattern:** A call to `PushEngage.onValueChanged(...)` that appears in any component OTHER than the immediate descendant of `AppRegistry.registerComponent`. Common bad locations: a screen's `useEffect` (post-navigation), inside a `FutureBuilder`-equivalent loader, behind auth gating, inside a deeply-nested `useEffect`.

**Categorize:** Worth fixing (cold-boot taps are lost; warm taps still work).

**References:** RN-BP-02, KI-RN003.

**Fix:** move the call to the ROOT component's `useEffect` (empty deps). Quote `init-code.md` Section 1 (deep-link listener block).

---

## RN-AUDIT-13 — Hardcoded App ID UUID in source

**Files to check:** `index.js` (and anywhere `setAppId` appears).

**Pattern:** `setAppId(...)` argument is a UUID string literal (not loaded from `react-native-config`, `react-native-dotenv`, build-time `--define`, or similar).

**Categorize:** Cosmetic.

**References:** XP-BP-01.

**Fix:** move UUIDs to env-based config (e.g., `react-native-config`'s `Config.PUSHENGAGE_APP_ID_IOS`). The App ID UUID is technically not a secret (transmitted in every API call as a public identifier), but the XP-BP-01 hygiene applies for dev/test/prod environment management.

---

## RN-AUDIT-14 — RN version vs SDK version compatibility

**Files to check:** `package.json`.

**Pattern:** `package.json` contains:
- `"@pushengage/pushengage-react-native": "^1.0.0"` (or any modern pin `>= 0.0.2`) AND
- `"react-native": "0.77.x"` or older (any version `< 0.78.0`).

**Categorize:** Likely cause (if iOS build fails with TurboModuleRegistry errors).

**References:** KI-RN009.

**Fix:** upgrade React Native to `0.78.0` or later (required). Every published release from `0.0.2` onward declares a `react-native >= 0.78.0` peer-dependency floor. The only pre-floor version on npm, `0.0.1` (2025-02, peer dep `react-native: *`), predates the modern API surface and is not a downgrade workaround — it's unsupported.

---

## RN-AUDIT-15 — Missing `apply plugin: "com.google.gms.google-services"`

**Files to check:** `android/app/build.gradle` (or `build.gradle.kts`).

**Pattern:** The app-level build.gradle does NOT contain `apply plugin: "com.google.gms.google-services"` (or `id("com.google.gms.google-services")` in plugins block) AND `android/app/google-services.json` exists.

**Categorize:** Likely cause (FCM token issuance silently fails; Android push never arrives).

**References:** RN-BP-11, KI-RN010.

**Fix:** add `apply plugin: "com.google.gms.google-services"` near the top of `android/app/build.gradle`, AFTER `apply plugin: "com.android.application"`.

---


## RN-AUDIT-17 — Cold-boot recovery missing (`getInitialNotification` not called)

**Files to check:** root component (the one passed to `AppRegistry.registerComponent`, typically `App.tsx`) + `index.*`.

**Pattern:** `PushEngage.onValueChanged(...)` is subscribed but `PushEngage.getInitialNotification()` is never called anywhere.

**Categorize:** Worth fixing (iOS cold-boot taps are lost; warm taps still work).

**References:** RN-BP-13, KI-RN003.

**Fix:** call `getInitialNotification()` once in the root startup effect alongside `onValueChanged` and route a non-null result. New in 1.0.0 — Android resolves `null`, so the call is cross-platform-safe. Quote `init-code.md` deep-link listener block.

---

## RN-AUDIT-18 — `setSmallIconResource` result compared to empty string

**Files to check:** all `*.js` / `*.ts` / `*.tsx`.

**Pattern:** `if (result === "")` / `if (result === '')` (or equivalent equality check) against a `setSmallIconResource` result.

**Categorize:** Worth fixing (the iOS resolution changed from `""` to `null` in 1.0.0; `=== ""` now silently never matches).

**References:** KI-RN012.

**Fix:** use `if (result == null)` / `if (!result)` instead of comparing to the empty string.

---

## Categorization summary

| Category | When to use |
|---|---|
| **Likely cause** | The finding explains a symptom the customer is reporting RIGHT NOW. |
| **Worth fixing** | Will cause a problem under predictable conditions (cold-boot tap lost, denied user can't recover). Not necessarily the current bug. |
| **Cosmetic** | No functional impact under current usage; worth addressing before release / scale-up. |

When reporting findings, present **Likely cause** first, then **Worth fixing**, then **Cosmetic** — and explicitly note which checks PASSED. The customer needs to know what was checked and ruled out.
