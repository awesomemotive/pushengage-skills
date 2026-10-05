# React Native SDK Version Matrix

Single source of truth for RN-SDK-version-pinned data. Other files in this spoke reference this one.

## Current stable version

**npm package (`@pushengage/pushengage-react-native`):** `1.1.0` — pinned via `^1.1.0` in `package.json`.

> **Release note (2026-10-02):** `1.1.0` adds **In-App Messaging** — `triggerIAMEvent(...)`, the `onIAMCustomAction` event emitter, and the exported `IAMCustomAction` type — and bumps the transitive natives to iOS `PushEngage 1.1.0` / Android `1.0.0`. No JS init change, no manifest/Info.plist/AppDelegate/MainActivity change; the native custom-action forwarder is enabled automatically on module load. **Breaking (iOS):** the native pod now requires iOS 15.0 — moot for RN apps, whose 0.78 baseline is already 15.1. See `in-app-messaging.md`. (History: `1.0.1` was a maintenance release over `1.0.0` — version bumps, example app on RN 0.86, codegen-compat typing for `trackEvent`; no public API change.)

**Native dependencies pulled transitively** (do NOT add these directly in the customer's main app Podfile target or Android dependencies block):

| Layer | Coordinate | Version |
|---|---|---|
| iOS pod (transitive via the plugin) | `PushEngage` | `1.1.0` (pinned exactly) |
| Android library (transitive via the plugin) | `com.github.awesomemotive:pushengage-android-sdk` | `1.0.0` |

> NOTE: on iOS the plugin's transitive `PushEngage 1.1.0` is the app-side pod; extension targets (NSE/NCE) link the separate extension-safe `PushEngageExtension 1.1.0` pod (see `install.md` / `nse-setup.md`). The transitive pin is build-authoritative — it's what `pod install` resolves.

## React Native compatibility split

| RN version | SDK version to use |
|---|---|
| `>= 0.78` | `^1.1.0` (latest). Uses Turbo Modules / New Architecture. |
| `< 0.78` | **No supported SDK build — upgrade RN to 0.78+ (required).** Every published release from `0.0.2` onward declares a `react-native >= 0.78.0` peer-dependency floor (npm registry). The only exception, `0.0.1` (2025-02, peer dep `react-native: *`), predates the modern API surface and is unsupported. (KI-RN009 / RN-AUDIT-14.) |

The skill should not attempt a downgrade workaround for RN < 0.78 — the only pre-floor npm version (`0.0.1`) lacks the modern API and is unsupported. The only supported path is to upgrade RN to 0.78+.

## Public TypeScript API surface (per version)

All members are properties on the default-exported `PushEngage` object. Async methods return `Promise<T>` and **throw** on failure — no `Result<T>` wrapper (unlike Flutter). Catch with `try/catch`. Rejections are plain `Error`s carrying a string `.code` and `.message` (the native side does not throw a typed `PushEngagePermissionError` — that class is not re-exported from the package, so it is not available at runtime).

### Setup

| Method | Signature | Notes |
|---|---|---|
| `setAppId` | `(appId: string) => void` | Call from `index.js` BEFORE `AppRegistry.registerComponent` (RN-BP-01). |
| `setEnvironment` | `(environment: 'STAGING' \| 'PRODUCTION') => void` | Switches between staging and production backends. **Call BEFORE `setAppId`** — both platforms cache base URLs on builder init. Case insensitive natively; unknown values default to production. |
| `getSdkVersion` | `() => string` | Returns the RN plugin version (`1.1.0` in source). **Unlike iOS native (KI-005), this works on RN.** |
| `setSmallIconResource` | `(resourceName: string) => Promise<void>` | Android-only; on iOS a no-op. Both platforms resolve `null` at runtime (iOS was `""` pre-1.0 — KI-RN012). Treat the result as nullable/falsy — the TS type is `Promise<void>`, so don't inspect it. |
| `enableLogging` | `(shouldEnable: boolean) => void` | Gate behind `if (__DEV__)` (RN-BP-05). |

### Badge

| Method | Signature | Notes |
|---|---|---|
| `setBadgeCount` | `(count: number) => void` | iOS uses `UNUserNotificationCenter.setBadgeCount` (16+) with `applicationIconBadgeNumber` fallback; Android badges subsequent notifications. Pass `0` to clear. NaN/±Infinity/out-of-Int-range → `0` (badge cleared). |

### Permissions

| Method | Signature | Notes |
|---|---|---|
| `requestNotificationPermission` | `() => Promise<boolean>` | Resolves with the actual grant outcome. **On grant, the SDK auto-calls `subscribe()`** — do not double-call. Rejects on failure with a plain `Error` carrying a string `.code` and `.message`; catch with `try/catch`. |
| `getNotificationPermissionStatus` | `() => string` (synchronous) | Returns `"granted"`, `"denied"`, OR (iOS-only) `"notYetRequested"`. **iOS has a third "not yet requested" state that Flutter/Android don't have** — handle it in switch logic. |

### Subscription

| Method | Signature | Notes |
|---|---|---|
| `subscribe` | `() => Promise<void>` | Throws on failure (no return value to inspect). |
| `unsubscribe` | `() => Promise<void>` | Throws on failure. |
| `getSubscriptionStatus` | `() => Promise<boolean>` | Local + remote check. |
| `getSubscriptionNotificationStatus` | `() => Promise<boolean>` | Combined subscribed-AND-permission-granted. |

### Subscriber identity

| Method | Signature | Notes |
|---|---|---|
| `getSubscriberId` | `() => Promise<string \| null>` | The subscriber hash. PII (RN-BP-07). |
| `getSubscriberDetails` | `(values: string[] \| null) => Promise<{ [k: string]: string \| number \| boolean } \| null>` | |
| `getDeviceTokenHash` | `() => Promise<string \| null>` | Android-only; resolves with `null` on iOS. |
| `addProfileId` | `(profileId: string) => Promise<string \| null>` | |
| `identify` | `(fields: IdentifyFields) => Promise<void>` | 12 predefined fields, Web SDK parity. Identical payload short-circuits locally. |
| `logout` | `(fieldNames: string[] \| null) => Promise<void>` | `null`/empty → default PII set. Rejects `TypeError` if the array contains a non-string (JS-boundary guard — KI-RN013 / breaking). |

### Attributes

| Method | Signature | Notes |
|---|---|---|
| `getSubscriberAttributes` | `() => Promise<{ [k: string]: string \| number \| boolean } \| null>` | |
| `addSubscriberAttributes` | `(attributes: { [k: string]: string }) => Promise<string \| null>` | Merge-style. |
| `setSubscriberAttributes` | `(attributes: { [k: string]: string }) => Promise<string \| null>` | Replaces ALL attributes. |
| `deleteSubscriberAttributes` | `(attributes: string[]) => Promise<string \| null>` | |

### Segments

| Method | Signature | Notes |
|---|---|---|
| `addSegment` | `(segments: string[]) => Promise<string \| null>` | |
| `removeSegment` | `(segments: string[]) => Promise<string \| null>` | |
| `addDynamicSegment` | `(segments: { name: string; duration: number }[]) => Promise<string \| null>` | Shape `{ name, duration }`; duration in days. |

### Events / campaigns

| Method | Signature | Notes |
|---|---|---|
| `sendTriggerEvent` | `(trigger: { [k: string]: string }) => Promise<string \| null>` | Pass a map; consider building from `TriggerCampaign` class's `toMap()`. |
| `sendGoal` | `(goal: { [k: string]: string \| number }) => Promise<string \| null>` | `{ name, count, value }`. |
| `addAlert` | `(alert: { [k: string]: string \| number \| undefined }) => Promise<string \| null>` | |
| `trackEvent` | `(event: TrackEventPayload) => Promise<void>` | Custom analytics. `eventName` required; `provider` defaults `"PushEngage"`, `eventType` defaults `"PushEngage.CustomEvent"`. |
| `automatedNotification` | `(status: boolean) => Promise<string \| null>` | **`boolean`, not enum** — unlike Flutter. The exported `TriggerStatusType` enum is unused by this method. |

### In-App Messaging (new in 1.1.0)

| Method | Signature | Notes |
|---|---|---|
| `triggerIAMEvent` | `(eventName: string, parameters?: { [k: string]: string \| number \| boolean }) => Promise<string \| null>` | Custom IAM trigger. Resolves with the native message (`"In-app message event triggered successfully"`) — means the event was accepted, NOT that a message was shown. Empty name → rejects with `.code` `"400"`, message `Event name is required`. Omitted `parameters` is coerced to `null` at the codegen boundary for you. Values are stringified natively before matching; Android drops `null` values. Works without a push subscription. See `in-app-messaging.md`. |
| `onIAMCustomAction` | `EventEmitter<IAMCustomAction>` — subscribe via `PushEngage.onIAMCustomAction(cb)`, returns `EventSubscription` | Fires for `custom` IAM button taps only. Native forwarding is auto-enabled on module load — no activation call. `subscription.remove()` in the `useEffect` cleanup (RN-BP-14). Not buffered for cold boot — by design. |

### Cold-boot / launch

| Method | Signature | Notes |
|---|---|---|
| `getInitialNotification` | `() => Promise<InitialNotification \| null>` | iOS: returns the cold-boot tap once, then `null` — the ONLY way to recover a force-quit launch tap (routed exclusively here, never replayed through `onValueChanged`). Android: always `null` (cold-boot deep links arrive through the host activity intent filter). Pair with `onValueChanged` (RN-BP-13). |

### Firebase config (Android)

| Method | Signature | Notes |
|---|---|---|
| `runConfigValidation` | `(senderId: string, projectId: string) => Promise<boolean>` | Android-only; resolves `true` when local config matches the supplied sender/project id. iOS no-op resolves `true`. |

### Event listeners

| Member | Signature | Notes |
|---|---|---|
| `onValueChanged` | `EventEmitter<{ deepLink: string; data: { [k: string]: string } }>` | Turbo Module event emitter. Subscribe via `PushEngage.onValueChanged(callback)` — returns `EventSubscription`. **Call `.remove()` from `useEffect` cleanup** (RN-BP-08). |
| `onFcmConfigError` | `EventEmitter<{ code: number; message: string }>` | Android-only FCM config error stream (codes `5001` sender-id mismatch / `5002` project-id mismatch / `5003` local-config invalid). Auto-enabled on module load; subscribe via `PushEngage.onFcmConfigError(cb)`, remove via `subscription.remove()`. Never fires on iOS. |

## Helper classes defined internally but NOT re-exported

These are defined internally but are **not re-exported from the package entry** — the package entry only re-exports the default `PushEngage` object plus the types `Environment` / `InitialNotification` / `FcmConfigError` / `IdentifyFields` / `TrackEventPayload`. They are therefore **not importable** by consumers from `@pushengage/pushengage-react-native` — treat them as internal-only.

| Type | Shape |
|---|---|
| `TriggerCampaign` | Constructor: `{ campaignName, eventName, referenceId?, profileId?, data? }`. Has `toMap()`. **Not importable** — for `sendTriggerEvent`, the consumer passes a plain object/map directly. |
| `TriggerStatusType` (enum) | `.Enabled = 'enabled'`, `.Disabled = 'disabled'`. **Not importable** — and unused anyway: `automatedNotification` takes `boolean`, not this enum. |
| `PushEngagePermissionError` (Error subclass) | Has `code?: string`. **Not importable, and not thrown at runtime** — the native side rejects permission methods with a plain `Error` carrying a string `.code`/`.message`. No `PushEngagePermissionError` instance exists at runtime. |

## Exported TypeScript types

New in 1.0.0. Importable from `@pushengage/pushengage-react-native`.

| Type | Definition |
|---|---|
| `Environment` | `type Environment = 'STAGING' \| 'PRODUCTION'` (the `setEnvironment` argument type) |
| `InitialNotification` | `interface { deepLink: string; data: { [k: string]: string } }` |
| `FcmConfigError` | `interface { code: number; message: string }` |
| `IdentifyFields` | `interface` with 12 optional keys `first_name, last_name, email, phone, gender, dob, language, profile_id, country, city, state, zip`, each `string \| number \| boolean` |
| `TrackEventPayload` | `interface { eventName: string; data?: {...}; profileId?: string; provider?: string; eventType?: string }` |
| `IAMCustomAction` | **NEW (1.1.0).** `interface { actionId: string; parameters: { [k: string]: string } }` — the `onIAMCustomAction` payload. **Route on `parameters.action`** (the dashboard Action name); `actionId` is a dashboard-generated internal key (e.g. `t04B05`). `parameters` values are always strings (stringified natively). |

## Removed from the public API

`setFcmConfigErrorListenerEnabled` was removed from the public `PushEngage` object in 1.0.0. Forwarding is auto-enabled on module load; the native/codegen method remains internal-only. Don't reference it in customer code — `PushEngage.setFcmConfigErrorListenerEnabled(...)` now throws `undefined is not a function`.

The same pattern applies to `setIAMCustomActionHandlerEnabled` (1.1.0): it exists on the native codegen spec, is called once by the module on load, and is **not** on the public `PushEngage` object — `PushEngage.setIAMCustomActionHandlerEnabled(...)` throws `undefined is not a function` (KI-RN015 / RN-AUDIT-20). Subscribe to `onIAMCustomAction` directly.

## Turbo Module name

- Registered as `PushengageReactNative` via `TurboModuleRegistry.getEnforcing<Spec>('PushengageReactNative')`.
- Useful for log filtering / native bridge debugging.

## Minimum platform versions

| Layer | Minimum |
|---|---|
| React Native | `0.78.0` (enforced via `react-native: >=0.78.0` in the plugin's `peerDependencies`). |
| Node.js | `18+` (per `.nvmrc`: v18; README states 18+). The SDK's example app workspace alone requires `>= 22.11.0` (its RN 0.86 baseline as of 1.0.1) — not a requirement for customer apps. |
| iOS deployment target | `15.1+` (the RN 0.78 baseline). **The SDK example app forces `IPHONEOS_DEPLOYMENT_TARGET = '16.0'` for the PushEngage/PushEngageExtension pods (a Swift-toolchain link requirement on Xcode 26), and `setBadgeCount` uses an iOS 16+ API with a pre-16 fallback** — so for badge usage treat 16 as the practical floor. |
| Android `minSdk` | `21` (Android 5.0+). Example app uses `24`. `compileSdk`/`targetSdk` = `31`, Kotlin `2.1.20`. |
| Architecture | **New Architecture (Turbo Modules) required.** Set `ENV['RCT_NEW_ARCH_ENABLED'] = '1'` in `ios/Podfile` and `newArchEnabled=true` in `android/gradle.properties`. |

## Companion library pins (Android side)

The customer adds these — the RN plugin does NOT bring Firebase transitively.

| Library | Recommended version | Coordinate |
|---|---|---|
| `google-services` Gradle plugin | `4.4.2` | `com.google.gms:google-services:4.4.2` (project-level classpath). The example app also uses `4.4.2` as of 1.0.1 (it used `4.3.15` in 1.0.0). |
| Firebase BOM | `33.5.1` | `platform('com.google.firebase:firebase-bom:33.5.1')` |
| Firebase Messaging | (from BOM) | `com.google.firebase:firebase-messaging` |

## Stale public-doc items (use for audit findings)

Some earlier setup guides show these older values. Use this matrix as the source of truth.

| Public doc value | Actual current value | Audit ref |
|---|---|---|
| `import PushEngage from 'pushengage-react-native'` | `from '@pushengage/pushengage-react-native'` | RN-AUDIT-01 / KI-RN001 |
| `pod 'PushEngage', '~>0.0.5'` in NSE Podfile snippet | `pod 'PushEngageExtension', '1.1.0'` | RN-AUDIT-03 / KI-RN011 |
| `pod 'PushEngage', '0.0.5'` in NCE Podfile snippet | `pod 'PushEngageExtension', '1.1.0'` | RN-AUDIT-03 |
| NSE deployment target "iOS 10+" | iOS 15.1+ (the RN 0.78 baseline) | — |

## SDK example app gaps (do NOT copy verbatim)

The SDK's own example app at `https://github.com/awesomemotive/pushengage-react-native/tree/master/example` (the repo's default branch is `master`, not `main`) has two known gaps. When citing the example as a reference, flag these:

| Gap | Fix |
|---|---|
| `AndroidManifest.xml` missing `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` | Add it. (KI-RN004 / RN-AUDIT-05.) |
| The example app's `index.js` calls `PushEngage.enableLogging(true)` unconditionally | Wrap in `if (__DEV__) { ... }`. (RN-BP-05 / RN-AUDIT-11.) |

## Package channels

- **npm:** `https://www.npmjs.com/package/@pushengage/pushengage-react-native`. Source of truth for published version.
- **Source code:** `https://github.com/awesomemotive/pushengage-react-native`. The repo's published TypeScript source is the canonical API reference.
- **iOS pod:** transitively via the plugin (`s.dependency 'PushEngage', '1.0.0'`). NSE/NCE targets additionally link the extension-safe `PushEngageExtension` pod at the same version.
- **Android library:** transitively via the plugin (`pushengage-android-sdk:1.0.0`). JitPack-hosted.

## Known-issue version bounds

Maintained in `pushengage-debug/known-issues.md`. RN entries use IDs `KI-RN001..KI-RN015`. Each row references this matrix's `1.1.0` baseline (with underlying native pins iOS `1.1.0` / Android `1.0.0`). Added at `1.0.1`: `KI-RN012` (the `setSmallIconResource` iOS resolution change `""`→`null`), `KI-RN013` (the `logout` `TypeError` JS-boundary guard), and `KI-RN014` (the `onFcmConfigError` / `runConfigValidation` iOS-asymmetry — both are Android-only no-ops on iOS).

---

## When the RN plugin version bumps

1. Bump this file's "Current stable version" section.
2. Bump the `package.json` snippet in `install.md`.
3. Re-verify the API surface table — read the plugin's published TypeScript source.
4. Check if transitive native pins changed (`s.dependency 'PushEngage'`; `implementation 'com.github.awesomemotive:pushengage-android-sdk'`).
5. Add or close known-issue rows in `pushengage-debug/known-issues.md` (KI-RN0NN entries).
6. Update `references.md` if the GitHub release tag URL changes.
7. Re-verify the RN version compatibility split — if the plugin drops support for RN < 0.78 entirely, update the compatibility table.
8. Re-verify the IAM surface (`triggerIAMEvent`, `onIAMCustomAction`, `IAMCustomAction`) against `in-app-messaging.md` and audit checks RN-AUDIT-19/20.
