---
name: pushengage-react-native
description: Use ONLY when the active repo is a React Native app (`package.json` with `react-native` dep, typically alongside `ios/` and `android/` folders, Turbo Modules / New Architecture). Handles full PushEngage integration end-to-end (JS init + Android native config + iOS native config + NSE, plus in-app messaging) and debugging existing integrations. Do not use for native iOS, native Android, or Flutter apps even if PushEngage is mentioned.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.1.0"
---

# PushEngage — React Native Spoke

You're working in a React Native app. This skill walks the customer through integrating PushEngage from zero to verified test push, or debugs a broken integration.

> **Trigger gating.** This spoke fires only when `package.json` has `react-native` (or `expo`) as a dependency. The presence of `ios/Podfile` or `android/build.gradle` alone (without `package.json`) does NOT route here — that's the native iOS or Android spoke. The presence of `pubspec.yaml` routes to Flutter, not here. The hub's detection logic puts both Flutter and React Native AHEAD of native iOS/Android checks because RN/Flutter projects also contain `ios/` and `android/` folders.

## Routing

| Customer intent | Read this file next |
|---|---|
| "Integrate PushEngage" / "Set up PushEngage" (greenfield or partial) | This file's "Integration flow" section below. |
| "PushEngage isn't working" / specific symptom | Run static audit (`audit-checks.md`) first, then escalate to `../pushengage-debug/SKILL.md` for Phase 2/3 (skill: `pushengage-debug`). |
| "Audit my PushEngage setup" | Read `audit-checks.md` and run the checks. |
| "Set up / add in-app messages", "trigger an in-app message", "in-app message never shows", custom action buttons | Read `in-app-messaging.md` (JS API), then `../pushengage/in-app-messaging.md` for the platform-independent semantics. Requires `^1.1.0`. |
| Concept / terminology question | Read `../pushengage/concepts.md`. |
| Best practices / rule lookup | Read `best-practices.md` (RN-specific) or `../pushengage/best-practices.md` (cross-platform). |
| API surface / "does method X exist" | Read `version-matrix.md` first; if not listed, fetch only URLs from `references.md`. |

## Integration flow (greenfield)

Five parts in order. Earlier artifacts feed later steps.

### Part 1 — Discovery & prereqs

Before touching anything, gather:

1. **Target platforms** — does the customer ship to both stores, or just one? Look for `ios/` and `android/` folders.
2. **React Native version** — read from `package.json`'s `dependencies."react-native"`. **Critical compatibility check:**
   - RN `>= 0.78` → install latest SDK (`^1.1.0`).
   - RN `< 0.78` → upgrade RN to 0.78+ (required — no supported SDK build for older RN; the Turbo Modules API requires it). See `version-matrix.md`.
3. **Existing SDK pin** — does `package.json` already have `@pushengage/pushengage-react-native`? If so, what version?
4. **`applicationId` (Android)** — read from `android/app/build.gradle`'s `defaultConfig.applicationId`.
5. **iOS Bundle Identifier** — confirm from Xcode (Signing & Capabilities → Bundle Identifier on the main target).
6. **PushEngage account state** — ask:
   - Have you already created PushEngage sites for this app? (One per platform.)
   - Do you have a Firebase project (Android)?
   - Do you have an Apple Developer account with Push Notifications enabled (iOS)?
7. **Expo vs Bare:** if `package.json` has `expo` as a dep, flag: **Expo Go does not support custom native modules.** Customer must use Expo Dev Client (`expo-dev-client`) and run `expo prebuild` once to expose `ios/` and `android/`.

Output a one-paragraph integration plan in chat. Wait for the customer to confirm before any edits.

### Part 2 — Provider setup (out-of-code, before any code edits)

Do these BEFORE editing code. The dashboard's App ID UUIDs are needed for `setAppId(...)`; Firebase + APNs have real-world setup steps that block code progress.

- **Android target → `firebase-setup.md`.** Defers to `../pushengage-android/firebase-setup.md` for the click-path. Captures Sender ID, Service Account JSON, `google-services.json`, `applicationId`.
- **iOS target → `apns-setup.md`.** Defers to `../pushengage-ios/provider-setup.md` for the click-path. Captures `.p12`, password, Bundle Identifier.

### Part 3 — Dashboard setup

Read `dashboard-setup.md`. A cross-platform RN app needs TWO PushEngage sites (one Android, one iOS) — each generates its own App ID UUID. Capture BOTH UUIDs and confirm them back to the customer.

### Part 4 — In-code edits

#### 4a — Install + native config

Read `install.md`. Apply:

- `package.json` — install via `npm install @pushengage/pushengage-react-native` (or yarn). **If RN < 0.78, upgrade RN to 0.78+ first — no supported SDK build for older RN.**
- Android: `android/gradle.properties` (`newArchEnabled=true`), `android/build.gradle` (google-services classpath + jitpack repo), `android/app/build.gradle` (**MUST `apply plugin: "com.google.gms.google-services"`** in the app module — the buildscript classpath alone is not enough), `AndroidManifest.xml` (`POST_NOTIFICATIONS`), `google-services.json` placement.
- iOS: `ios/Podfile` (`ENV['RCT_NEW_ARCH_ENABLED'] = '1'`, NSE/NCE target blocks with `pod 'PushEngageExtension', '1.1.0'` — the 1.0.0 two-pod split, no `APPLICATION_EXTENSION_API_ONLY` override needed), `Info.plist` (`PushEngage_App_Group_Key`, background modes), `cd ios && pod install`.

**Auto-edit + announce posture** for config files. Show the diff, then apply.

#### 4b — JS + ObjC init code

Read `init-code.md`. Apply:

- `index.js` / `index.tsx` — import from `@pushengage/pushengage-react-native` (NOT `pushengage-react-native`), `PushEngage.setAppId(...)` with `Platform.OS === 'ios' ? ... : ...` BEFORE `AppRegistry.registerComponent`, `if (__DEV__) PushEngage.enableLogging(true);`.
- iOS AppDelegate — swizzling enabled in the delegate's `init`, `setInitialInfo` in `didFinishLaunchingWithOptions` before React Native boots. RN ≥ 0.77 templates generate `AppDelegate.swift` (use the Swift deltas in `init-code.md` Section 2); upgraded apps may still have `AppDelegate.mm` (use the ObjC++ template).
- `onValueChanged` listener in root component's `useEffect` with `() => subscription.remove()` cleanup.
- Register `getInitialNotification()` alongside `onValueChanged` in the root effect for iOS cold-boot tap recovery (RN-BP-13). New in 1.0.0; resolves `null` on Android.
- **No Android `MainApplication.kt` PushEngage init needed** — RN plugin handles native init.

**Diff + confirm posture** for `index.js`, the AppDelegate (`.swift` or `.mm`), and the root `App.tsx` (when adding the listener).

Insert the App ID UUID(s) captured in Part 3.

#### 4c — Runtime permission flow

Read `permission-flow.md`. Set up `requestNotificationPermission()` behind a user action (button, end-of-onboarding) — NOT in `index.js` or root `useEffect`. SDK auto-calls `subscribe()` on grant; don't double-call.

For Android: confirm `POST_NOTIFICATIONS` is in Manifest (Step 4a).

#### 4d — NSE setup (iOS, required for rich notifications)

If iOS target: read `nse-setup.md`. Walk through Xcode NSE target creation, deployment target iOS 15.1, Podfile target block with `pod 'PushEngageExtension', '1.1.0'` (the extension-safe pod from the 1.0.0 two-pod split, NOT `pod 'PushEngage', '~>0.0.5'`), App Groups capability on BOTH main and NSE targets, matching `PushEngage_App_Group_Key`.

**Skip entirely if Android-only.**

#### 4e — NCE setup (optional)

Default: SKIP. Read `nce-setup.md` if customer asks for custom in-notification UI.

#### 4f — In-App Messaging — OPTIONAL

Read `in-app-messaging.md` only if the customer asks for in-app messages (or already has IAM campaigns on the dashboard). Default: skip; mention it exists in one line. IAM needs nothing beyond `setAppId` in Part 4b — no native edits, no activation call. Optional code: an `onIAMCustomAction` subscription next to `onValueChanged` in the root `useEffect` (diff + confirm) and `triggerIAMEvent` calls at the product moments the campaigns key off. Verification does **not** need a real device.

### Part 5 — Verification

Per target platform:

1. Real device:
   - **Android:** must have Google Play Services. NOT a Play-Services-less emulator.
   - **iOS:** must be a physical device. APNs does NOT work on simulator.
2. Install + run + trigger permission request → grant.
3. PushEngage dashboard → corresponding site → Push Broadcasts → send test → confirm arrival.

If verification fails: run static audit (`audit-checks.md`), then escalate to `pushengage-debug/SKILL.md`.

### Part 6 — Post-integration best-practices nudge

Short message. Pick 2–3 rules applicable to what you integrated:

- XP-BP-01 — move App ID UUID(s) out of source (`react-native-config`, `--env`).
- RN-BP-04 — defer `requestNotificationPermission()` to a meaningful moment.
- RN-BP-05 — wrap `enableLogging(true)` in `__DEV__`.
- RN-BP-02 — confirm `onValueChanged` is in the root component's `useEffect`, won't drift to a downstream screen during future refactors.
- RN-BP-13 — register `getInitialNotification()` alongside `onValueChanged` for iOS cold-boot tap recovery.
- ANDROID-BP-13 (if Android target) — document OEM battery optimization in customer's FAQ.

## Debugging entry points

Escalate to `pushengage-debug/SKILL.md` when:

- "Module 'pushengage-react-native' could not be resolved" → RN-AUDIT-01 / KI-RN001 (stale import name).
- "iOS build fails with 'Application extensions cannot use PushEngage'" → RN-AUDIT-03 (extension target links `pod 'PushEngage'` instead of `pod 'PushEngageExtension', '1.1.0'`). The old `APPLICATION_EXTENSION_API_ONLY` override / KI-RN002 path is obsolete in 1.0.0 (two-pod split).
- "iOS build fails with '@import' / c++ modules disabled" → KI-RN008. Note: the `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES` override this depended on is obsolete in 1.0.0 (two-pod split); the app target's `@import PushEngage` resolves without it.
- "Rich-push image doesn't load" → RN-AUDIT-03 (stale pod pin) and RN-AUDIT-09 (App Groups).
- "Cold-boot deep link is lost" → RN-AUDIT-12 (listener registered too late) / RN-AUDIT-17 (missing `getInitialNotification`) / KI-RN003. 1.0.0 provides the `getInitialNotification()` recovery API for iOS.
- "Works on Android but not iOS" (or vice versa) → RN-AUDIT-07 / KI-RN006.
- "Permission dialog never shows on Android" → RN-AUDIT-05 / KI-RN004.
- "Android push never arrives, FCM token never registers" → RN-AUDIT-15 / KI-RN010 (missing google-services plugin apply).
- "iOS build crashes with Turbo Module error after `npm install`" → RN-AUDIT-14 / KI-RN009 (RN/SDK version mismatch).
- "`PushEngage.triggerIAMEvent is not a function`" → stale `1.0.x` install; bump to `^1.1.0`, reinstall, `pod install`.
- "`PushEngage.setIAMCustomActionHandlerEnabled is not a function`" → RN-AUDIT-20 / KI-RN015 (internal method; subscribe to `onIAMCustomAction` directly).
- "In-app message never appears" / "in-app button does nothing" / "`onIAMCustomAction` never fires" → diagnostic tree Branch 8 + `in-app-messaging.md`.

## Edit posture per file class

- **Auto-edit + announce:** `package.json`, `ios/Podfile`, `android/build.gradle`, `android/settings.gradle`, `android/app/build.gradle`, `android/gradle.properties`, `AndroidManifest.xml`, `Info.plist`, NSE `Info.plist`, `google-services.json` placement.
- **Diff + confirm:** `index.js` / `index.tsx`, `ios/<App>/AppDelegate.mm`, root `App.tsx` (when adding listener). Entry-point code.
- **Customer-driven (you instruct):** Xcode UI — NSE/NCE target creation, App Groups capability, Push Notifications + Background Modes capabilities. Don't `.pbxproj`-edit.
- **Never touch:** anything outside integration scope. No formatting passes, no RN version bumps, no Gradle bumps beyond required.

## Where the RN spoke borrows from other spokes

The RN spoke references content files from the iOS and Android spokes to avoid duplication. **By file path only** — you read the referenced content file but does NOT load the other spoke's `SKILL.md`.

| What | Where |
|---|---|
| Firebase Console click-path | `../pushengage-android/firebase-setup.md` |
| Apple Developer + Keychain click-path | `../pushengage-ios/provider-setup.md` |
| Long-form NCE template + category consistency rules | `../pushengage-ios/nce-setup.md` |
| Manual (non-swizzling) iOS forwards template | `../pushengage-ios/init-code.md` |

**Never** read the other spokes' `SKILL.md`s. Only their content files.

## What you must not do

- **Don't import from `'pushengage-react-native'`** (missing scope). Always `'@pushengage/pushengage-react-native'`. RN-AUDIT-01.
- **Don't invent PushEngage method names.** Check `version-matrix.md`. If unsure, fetch an allowlisted URL from `references.md`. If still unsure, say so.
- **Use `version-matrix.md` as the source of truth for the package name and pod pins** — scoped import, and `PushEngageExtension 1.1.0` for NSE/NCE targets.
- **Don't copy the SDK's example app verbatim.** It has two known gaps: missing `POST_NOTIFICATIONS` in Manifest, unconditional `enableLogging(true)`. (It DOES apply `com.google.gms.google-services` — that step is still required in customer apps.) The skill's `install.md` and `init-code.md` cover the correct patterns.
- **Don't auto-edit `index.js` or `AppDelegate.mm` without showing the diff and getting confirmation.** Entry-point code is diff + confirm.
- **Don't touch files outside integration scope.**
- **Don't fetch URLs not in `references.md`.**
- **Don't ask the customer for Service Account JSON or `.p12` contents in chat.** Both are secrets — they upload directly to the dashboard.
- **Don't add `pod 'PushEngage'` to the main app target in Podfile.** Pulled transitively. NSE / NCE target blocks get `pod 'PushEngageExtension', '1.1.0'` (the extension-safe pod from the 1.0.0 two-pod split), not `pod 'PushEngage'`. RN-BP-06.
- **Don't add a `MainApplication.kt` PushEngage init block** — the plugin handles it.
- **Don't follow the SDK example app's unconditional `enableLogging(true)` pattern** — gate behind `__DEV__`.
- **Don't call `setIAMCustomActionHandlerEnabled` or `getIAMDatabaseState` from JS** (RN-BP-15) and don't call `setIAMCustomActionHandler` from native code — the module owns the native handler. Don't invent the dashboard click-path for creating IAM campaigns.
