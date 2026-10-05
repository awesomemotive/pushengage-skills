---
name: pushengage-flutter
description: Use ONLY when the active repo is a Flutter app (`pubspec.yaml` with a `flutter:` SDK block). Handles full PushEngage integration end-to-end (Dart init + Android native config + iOS native config + NSE, plus in-app messaging) and debugging existing integrations. Do not use for native iOS, native Android, or React Native apps even if PushEngage is mentioned.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.1.0"
---

# PushEngage — Flutter Spoke

You're working in a Flutter app. This skill walks the customer through integrating PushEngage from zero to verified test push, or debugs a broken integration.

> **Trigger gating.** This spoke fires only when `pubspec.yaml` contains a `flutter:` SDK block. The presence of `ios/Podfile` or `android/build.gradle` alone (without `pubspec.yaml`) does NOT route here — that's the native iOS or Android spoke. The hub's detection logic explicitly puts Flutter precedence ABOVE the native checks.

## Routing

| Customer intent | Read this file next |
|---|---|
| "Integrate PushEngage" / "Set up PushEngage" (greenfield or partial) | This file's "Integration flow" section below. |
| "PushEngage isn't working" / specific symptom | Run static audit (`audit-checks.md`) first, then escalate to `../pushengage-debug/SKILL.md` for Phase 2/3 (skill: `pushengage-debug`). |
| "Audit my PushEngage setup" | Read `audit-checks.md` and run the checks. |
| "Set up / add in-app messages", "trigger an in-app message", "in-app message never shows", custom action buttons | Read `in-app-messaging.md` (Dart API), then `../pushengage/in-app-messaging.md` for the platform-independent semantics. Requires plugin `^1.1.0` and `platform :ios, '15.0'`. |
| Concept / terminology question | Read `../pushengage/concepts.md`. |
| Best practices / rule lookup | Read `best-practices.md` (Flutter-specific) or `../pushengage/best-practices.md` (cross-platform). |
| API surface / "does method X exist" | Read `version-matrix.md` first; if not listed, fetch only URLs from `references.md`. |

## Integration flow (greenfield)

This is the happy path. Follow the parts strictly in order — earlier artifacts feed later steps.

### Part 1 — Discovery & prereqs

Before touching anything, gather:

1. **Target platforms** — does the customer ship to both stores (Android + iOS), or just one? Look for `ios/` and `android/` folders. Confirm with the customer (a Flutter project can have both folders even if they only deploy to one).
2. **Pub version pin** — read `pubspec.yaml`'s current `pushengage_flutter_sdk` pin (if any).
3. **`applicationId` (Android)** — grep `android/app/build.gradle*` for `applicationId`. Needed to register the Firebase app.
4. **iOS Bundle Identifier** — confirm with the customer; readable from Xcode's Signing & Capabilities tab on the Runner target.
5. **PushEngage account state** — ask:
   - Do you already have a PushEngage account?
   - Have you already created a site for this Flutter app? (One per platform — two sites total if shipping both.)
6. **Firebase project state** (Android-only step) — do they have a Firebase project for this app already, or do we create one?
7. **Apple Developer account state** (iOS-only step) — do they have it, and have they enabled Push Notifications on the App ID?
8. **Compose / UIKit / Material analog** — Flutter is the UI framework. No sub-flow split here (unlike Android's Compose-vs-Views split).

Output a one-paragraph integration plan in chat. Wait for the customer to confirm before any edits.

### Part 2 — Provider setup (out-of-code, before any code edits)

Do these BEFORE editing code. The dashboard's App ID UUIDs are needed for `setAppId(...)`; Firebase + APNs have real-world setup steps that block code progress.

The work splits by target platform:

- **Android target → `firebase-setup.md`.** Walks the customer through Firebase Console → register Android app → download `google-services.json` → generate Service Account JSON → capture Sender ID + `applicationId`. Defers to `../pushengage-android/firebase-setup.md` for the full click-path.
- **iOS target → `apns-setup.md`.** Walks the customer through Apple Developer portal → enable Push Notifications capability → Keychain CSR → APNs SSL cert → `.p12` export → capture Bundle Identifier. Defers to `../pushengage-ios/provider-setup.md` for the full click-path.

If both platforms: do them in parallel-ish — the steps don't conflict.

### Part 3 — Dashboard setup

Read `dashboard-setup.md`. A cross-platform Flutter app needs TWO PushEngage sites (one Android, one iOS) — each generates its own App ID UUID. Capture BOTH UUIDs and confirm them back to the customer before moving on.

Single-platform: just one site, one UUID.

### Part 4 — In-code edits

#### 4a — Install + native config

Read `install.md`. Apply:

- `pubspec.yaml` — bump or add `pushengage_flutter_sdk: ^1.1.0`. Run `flutter pub get`.
- Android: `android/settings.gradle`, `android/build.gradle`, `android/app/build.gradle`, `AndroidManifest.xml`, `google-services.json`.
- iOS: `ios/Podfile` (platform + post_install + NSE/NCE targets when applicable), `Info.plist`, `cd ios && pod install`.

**Auto-edit + announce posture** for all config files: `pubspec.yaml`, `Podfile`, `android/build.gradle*`, `settings.gradle*`, `AndroidManifest.xml`, `Info.plist`. Show the diff, then apply.

#### 4b — Dart + Swift init code

Read `init-code.md`. Apply:

- `lib/main.dart` — `PushEngage.setAppId(...)` in `main()` BEFORE `runApp`, with `Platform.isIOS ?` branching for cross-platform builds.
- `ios/Runner/AppDelegate.swift` — `swizzleInjection(isEnabled: true)` in `override init()`, optional `setBadgeCount(0)` and `setNotificationWillShowInForegroundHandler` in `didFinishLaunchingWithOptions`.
- Deep-link listener — `PushEngage.deepLinkStream.listen(...)` in the root widget's `initState`, cancelled in `dispose`.
- **No Android `Application` subclass needed** — the Flutter plugin handles native Android init through its plugin lifecycle.

**Diff + confirm posture** for `lib/main.dart` and `AppDelegate.swift` — entry-point code, bad edits break the app.

Insert the App ID UUID(s) captured in Part 3. Per XP-BP-01, recommend the customer later move them out of source (e.g., `--dart-define`).

#### 4c — Runtime permission flow

Read `permission-flow.md`. Set up a `requestNotificationPermission()` call site behind a user action (button, end-of-onboarding) — NOT in `main()` or app-root `initState`. Cross-platform via a single Dart call.

For Android: confirm `POST_NOTIFICATIONS` is declared in the Manifest (Step 4a covers this).

#### 4d — NSE setup (iOS, required for rich notifications)

If the customer is shipping to iOS, read `nse-setup.md`. Walk them through:

- Adding the NSE target via Xcode UI.
- Setting deployment target to iOS 15.
- Adding `pod 'PushEngageExtension', '1.1.0'` to the Podfile NSE target block (NOT to the `Runner` target).
- App Groups capability on BOTH targets + matching `PushEngage_App_Group_Key` in both Info.plists.

**Skip this step entirely if the customer is Android-only.**

#### 4e — NCE setup (optional)

Default: SKIP. Only run if the customer explicitly asks for custom in-notification UI. Read `nce-setup.md` if they do.

#### 4f — In-App Messaging — OPTIONAL

Read `in-app-messaging.md` only if the customer asks for in-app messages (or already has IAM campaigns on the dashboard). Default: skip; mention it exists in one line. IAM needs nothing beyond `setAppId` in Part 4b — no native edits. Optional code: an `onIAMCustomAction` subscription next to the `deepLinkStream` listener in the root widget (diff + confirm) and `triggerIAMEvent` calls at the product moments the campaigns key off. Verification does **not** need a real device.

### Part 5 — Verification

The success criterion is **a real test notification reaches the customer's device** (one per platform if cross-platform).

For each target platform:

1. Customer connects a real device:
   - **Android:** must have Google Play Services. NOT a Play-Services-less emulator.
   - **iOS:** must be a physical device. APNs does NOT work on the iOS simulator.
2. Customer installs and runs the app once. Triggers the in-app permission request (per Part 4c).
3. Customer goes to PushEngage dashboard → corresponding site → Campaign → Push Broadcasts → sends to test device (or "All Subscribers" if it's the only one).
4. Ask: "Did the notification arrive on `<platform>`?"

**If yes** → go to Part 6.
**If no** → run static audit first (`audit-checks.md`), then hand off to `pushengage-debug/SKILL.md` with the symptom "test push didn't arrive on `<platform>`."

### Part 6 — Post-integration best-practices nudge

Short message. Pick 2–3 rule IDs that actually apply to what you just integrated. Examples:

- XP-BP-01 — move the App ID UUID(s) out of source (`--dart-define`, `String.fromEnvironment`).
- FLUTTER-BP-04 — defer `requestNotificationPermission()` to a meaningful moment, not on launch.
- FLUTTER-BP-05 — wrap `enableLogging(true)` in `kDebugMode`.
- FLUTTER-BP-02 — confirm the deep-link listener is in the root widget's `initState` and won't drift to a downstream route during future refactors.
- ANDROID-BP-13 (if Android target) — document OEM battery-optimization caveat in the customer's user-facing FAQ.

(Customize to what you actually saw. Don't dump all of them.)

## Debugging entry points

If the customer says any of the following — or if Part 5 verification fails — escalate to `pushengage-debug/SKILL.md`:

- "Notifications aren't arriving" (on one or both platforms).
- "Permission prompt never shows" (Android-specific → check the MERGED manifest for `POST_NOTIFICATIONS`; the SDK's library manifest normally supplies it — see FLUTTER-AUDIT-05).
- "iOS build fails with 'Application extensions cannot use PushEngage'" → FLUTTER-AUDIT-04.
- "Rich-push image doesn't load" → FLUTTER-AUDIT-03 (stale pod pin) and FLUTTER-AUDIT-09 (App Groups mismatch).
- "Cold-boot deep link is lost" → FLUTTER-AUDIT-12 / KI-F003.
- "Works on Android but not iOS" (or vice versa) → FLUTTER-AUDIT-07 / KI-F006 (hardcoded App ID).
- "`flutter pub get` succeeded but `PushEngage.X()` says no such method" → FLUTTER-AUDIT-01 / KI-F001 (stale `^0.0.1`).
- "Gradle build fails: `google-services.json` missing" → FLUTTER-AUDIT-06.
- "`pod install` fails: pushengage_flutter_sdk (1.1.0) / PushEngage (1.1.0) does not support iOS 12.0–14.x" → FLUTTER-AUDIT-15 / KI-F013 (Podfile `platform :ios, '15.0'`).
- "In-app message never appears" / "in-app button does nothing" / "`onIAMCustomAction` never fires" → diagnostic tree Branch 8 + `in-app-messaging.md`.

The debug skill handles audit + known-issues + symptom tree.

## Edit posture per file class

- **Auto-edit + announce:** `pubspec.yaml`, `Podfile`, `android/build.gradle*`, `android/settings.gradle*`, `android/app/build.gradle*`, `AndroidManifest.xml`, `Info.plist`, NSE `Info.plist`, file placements (`google-services.json` move).
- **Diff + confirm:** `lib/main.dart`, `ios/Runner/AppDelegate.swift`. Entry-point code — bad edits break the app.
- **Customer-driven (you instruct only):** Xcode UI work — NSE/NCE target creation, App Groups capability toggles, Push Notifications capability, Background Modes. Don't `.pbxproj`-edit programmatically.
- **Never touch:** anything outside the integration scope. No formatting passes, no Flutter / Dart / Gradle version bumps beyond what's required, no opportunistic refactoring.

## Where the Flutter spoke borrows from other spokes

The Flutter spoke deliberately references content files from the iOS and Android spokes to avoid duplication. These cross-spoke references are by file path only — you read the referenced content file, but does NOT load the other spoke's `SKILL.md`. The trigger contract (Flutter session loads Flutter skill) is preserved.

| What | Where |
|---|---|
| Firebase Console click-path | `../pushengage-android/firebase-setup.md` (referenced from this spoke's `firebase-setup.md`) |
| Apple Developer portal + Keychain click-path | `../pushengage-ios/provider-setup.md` (referenced from this spoke's `apns-setup.md`) |
| Long-form NCE template + category consistency rules | `../pushengage-ios/nce-setup.md` (referenced from this spoke's `nce-setup.md`) |
| Manual (non-swizzling) iOS forwards template | `../pushengage-ios/init-code.md` (referenced from this spoke's `init-code.md`) |

**Never** read the other spokes' `SKILL.md`s. Only their content files.

## What you must not do

- **Don't invent PushEngage method names.** If unsure a Dart method exists, check `version-matrix.md`. If still unsure, fetch an allowlisted URL from `references.md`. If still unsure, say so — don't guess.
- **Use `version-matrix.md` as the source of truth for version pins** (`^1.1.0` Flutter, `1.1.0` iOS pods, `platform :ios, '15.0'`). Older guides may show earlier pins like `^0.0.1`.
- **Don't auto-edit `lib/main.dart` or `AppDelegate.swift` without showing the diff and getting confirmation.** Entry-point code is diff + confirm.
- **Don't touch files outside the integration scope.** No refactoring, no Dart version bumps, no Flutter SDK upgrades, no Gradle version bumps that weren't strictly required.
- **Don't fetch URLs not in `references.md`.** The allowlist exists so the customer can trust where you got information.
- **Don't ask the customer to paste Service Account JSON or `.p12` contents in chat.** Both are secrets. They upload directly to the PushEngage dashboard; you only need to know the local path and confirm the upload succeeded.
- **Don't add `pod 'PushEngage'` to the main `Runner` target in Podfile** — it's pulled transitively. Only add it to NSE / NCE target blocks. (FLUTTER-BP-06.)
- **Don't create an Android `Application` subclass for PushEngage init** — the Flutter plugin handles it. If one already exists for other reasons, leave it alone.
- **Don't call `setIAMCustomActionHandler` from `AppDelegate.swift`** — the plugin registers it; a host call overwrites it and the Dart `onIAMCustomAction` stream goes silent. Don't forward `onRequestPermissionsResult` in `MainActivity.kt` for IAM. Don't invent the dashboard click-path for creating IAM campaigns.
- **Prefer `setAppId` in `main()` before `runApp`** over the SDK example's `initState` placement. The example sets it from `initState` because it enters the App ID at runtime — a valid pattern, not a bug — but for a known App ID, `main()` is cleaner. (FLUTTER-BP-01.)
