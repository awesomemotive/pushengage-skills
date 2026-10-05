# Flutter SDK Version Matrix

Single source of truth for Flutter-SDK-version-pinned data. Other files in this spoke reference this one.

## Current stable version

**Flutter plugin (`pushengage_flutter_sdk`):** `1.1.0` — pinned via `^1.1.0` in `pubspec.yaml`.

> **Release note (2026-10-02):** `1.1.0` adds **In-App Messaging** — `triggerIAMEvent(...)`, the `onIAMCustomAction` stream, and the `IAMCustomAction` model — and bumps the transitive natives to iOS `PushEngage 1.1.0` / Android `1.0.0`. No Dart init change, no manifest/Info.plist/AppDelegate change. **Breaking (iOS):** the plugin podspec and native pods now require deployment target **iOS 15.0** (was 12.0) — Podfiles below `platform :ios, '15.0'` fail at `pod install` (KI-F013). Minimum Flutter SDK is now `>=3.22.2`. Also fixes plugin registration under the UIScene life cycle (iOS 27 SDK / Flutter 3.35+) — no integration change needed. See `in-app-messaging.md`.

**Native dependencies pulled transitively** (do NOT add these directly in the customer's Podfile `Runner` target or Android dependencies block):

| Layer | Coordinate | Version |
|---|---|---|
| iOS pod (app target, transitive via plugin's podspec) | `PushEngage` | `1.1.0` (pinned exactly, no `~>`) |
| iOS pod (extension targets, transitive via `PushEngage` → `PushEngageExtension`) | `PushEngageExtension` | `1.1.0` |
| Android library (transitive via the plugin) | `com.github.awesomemotive:pushengage-android-sdk` | `1.0.0` |

As of iOS native `1.0.0` the iOS pod is **split in two**: the app `Runner` target gets `PushEngage` (transitively, via the plugin's podspec), while the NSE/NCE extension targets get `PushEngageExtension` (the extension-safe pod, which `PushEngage` depends on). See `install.md`, `nse-setup.md`, and `nce-setup.md`.

## Public Dart API surface (per version)

All methods are `static` on the `PushEngage` class. Async methods return `Future<PushEngageResult<T>>` (typed result wrapper with `.data` and `.error`).

### Setup

| Method | `1.1.0` | Notes |
|---|---|---|
| `setAppId(String appId) → Future<void>` | ✅ | Call from `main()` AFTER `WidgetsFlutterBinding.ensureInitialized()` and BEFORE `runApp(...)`. (FLUTTER-BP-01.) Now also forwards the plugin version (`sdkVersion`) over the channel for attribution. |
| `setEnvironment(Environment environment) → Future<void>` | ✅ | **NEW.** Switches between `Environment.staging` and `Environment.production` backends. Must be called BEFORE `setAppId` — the native Android SDK caches its base URLs when the app id is set. Defaults to production; omit entirely for normal integrations. |
| `setBadgeCount(int count) → Future<void>` | ✅ | Pass `0` to clear. Values outside the 32-bit int range are coerced to `0`. iOS uses the system badge (iOS 16+ API with a fallback); Android applies it to subsequent notifications. |
| `getSdkVersion() → String` | ✅ | Returns `"1.1.0"` (the Flutter plugin version, NOT the underlying native SDK version). iOS native also ships `getSdkVersion` now (KI-005 fixed); cross-platform parity holds. |
| `setSmallIconResource(String resourceName) → Future<void>` | ✅ Android only | No-op on iOS (guarded by `Platform.isAndroid`). |
| `enableLogging(bool shouldEnable) → void` | ✅ sync | Toggles Dart-side `DebugLogger` and native logging. Gate behind `kDebugMode` (FLUTTER-BP-05). |

### Permissions

| Method | `1.1.0` | Notes |
|---|---|---|
| `requestNotificationPermission() → Future<PushEngageResult<bool>>` | ✅ | BOTH platforms return the actual grant boolean — the future resolves after the user responds to the prompt (already-decided permission resolves immediately with the current state). Errors are swallowed to `success(false)`, never a `failure`. `getNotificationPermissionStatus()` remains the way to poll the current state later. |
| `getNotificationPermissionStatus() → Future<PushEngageResult<String>>` | ✅ | Android returns `"granted"` or `"denied"`. iOS additionally returns `"notYetRequested"` when permission has not been requested yet. The Dart layer passes the native string through (defaults to `"denied"` only on null). |

### Subscription

| Method | `1.1.0` | Notes |
|---|---|---|
| `subscribe() → Future<PushEngageResult<bool>>` | ✅ | Auto-called on permission grant; rarely needs manual invocation. |
| `unsubscribe() → Future<PushEngageResult<bool>>` | ✅ | |
| `getSubscriptionStatus() → Future<PushEngageResult<bool>>` | ✅ | Local + remote check. |
| `getSubscriptionNotificationStatus() → Future<PushEngageResult<bool>>` | ✅ | Combined subscribed-AND-permission-granted check. |

### Subscriber identity

| Method | `1.1.0` | Notes |
|---|---|---|
| `getSubscriberId() → Future<PushEngageResult<String?>>` | ✅ | The subscriber hash. Treat as PII (FLUTTER-BP-07). |
| `getSubscriberDetails(List<String>? values) → Future<PushEngageResult<Map<String, dynamic>?>>` | ✅ | Now accepts `null`/empty to fetch all fields, and returns a **failure** when the user is not subscribed — callers must handle the failure path. |
| `identify(IdentifyFields fields) → Future<PushEngageResult<String?>>` | ✅ | Sends only the fields that are set. Native short-circuits identical repeat calls (24h TTL on iOS). PII (FLUTTER-BP-07). |
| `logout(List<String>? fieldNames) → Future<PushEngageResult<String?>>` | ✅ | `null`/empty clears the default PII set (`first_name`, `last_name`, `email`, `phone`, `gender`, `dob`, `profile_id`). |
| `getDeviceTokenHash() → Future<PushEngageResult<String?>>` | ✅ Android only | iOS returns `failure('Platform is not Android')`. (KI-F008.) |
| `addProfileId(String profileId) → Future<PushEngageResult<String?>>` | ✅ | Associates a host-app user ID with the subscriber. PII (FLUTTER-BP-07). |

### Attributes

| Method | `1.1.0` | Notes |
|---|---|---|
| `getSubscriberAttributes() → Future<PushEngageResult<Map<String, dynamic>>>` | ✅ | |
| `addSubscriberAttributes(Map<String, dynamic> attributes) → Future<PushEngageResult<String?>>` | ✅ | Replaces existing keys (merge-style). |
| `setSubscriberAttributes(Map<String, dynamic> attributes) → Future<PushEngageResult<String?>>` | ✅ | Replaces ALL attributes (clobber). |
| `deleteSubscriberAttributes(List<String> attributes) → Future<PushEngageResult<String?>>` | ✅ | |

### Segments

| Method | `1.1.0` | Notes |
|---|---|---|
| `addSegment(List<String> segments) → Future<PushEngageResult<String?>>` | ✅ | |
| `removeSegment(List<String> segments) → Future<PushEngageResult<String?>>` | ✅ | |
| `addDynamicSegment(List<DynamicSegment> segments) → Future<PushEngageResult<String?>>` | ✅ | |

### Events / campaigns

| Method | `1.1.0` | Notes |
|---|---|---|
| `sendTriggerEvent(TriggerCampaign trigger) → Future<PushEngageResult<String?>>` | ✅ | Parameter is named `trigger`. |
| `trackEvent(TrackEventPayload event) → Future<PushEngageResult<String?>>` | ✅ | `event.eventName` must be non-empty — `TrackEventPayload.toMap()` throws `ArgumentError` synchronously otherwise. Native defaults `provider`/`eventType` when omitted. |
| `sendGoal(Goal goal) → Future<PushEngageResult<String?>>` | ✅ | |
| `addAlert(TriggerAlert alert) → Future<PushEngageResult<String?>>` | ✅ | `expiryTimestamp` now serializes as UTC ISO-8601 truncated to millisecond precision (`...HH:mm:ss.SSS` + `Z`). |
| `automatedNotification(TriggerStatusType status) → Future<PushEngageResult<String?>>` | ✅ | `status` is enum `TriggerStatusType.enabled` / `.disabled`. |

### In-App Messaging (new in 1.1.0)

| Method | `1.1.0` | Notes |
|---|---|---|
| `triggerIAMEvent(String eventName, [Map<String, dynamic>? parameters]) → Future<PushEngageResult<String?>>` | ✅ NEW | Custom IAM trigger. Positional optional `parameters`; values should be `String`/`num`/`bool` (other types diverge per platform: Android `toString()`s them, iOS drops the key). Success `data` = native message string (`"In-app message event triggered successfully"`) — means the event was accepted, NOT that a message was shown. Empty name → `failure` wrapping a `PlatformException` with code `400`, message `Event name is required`. Values are stringified natively before matching; Android drops `null` values. Works without a push subscription. See `in-app-messaging.md`. |
| `static Stream<IAMCustomAction> get onIAMCustomAction` | ✅ NEW | Broadcast stream of `custom` IAM button taps. Lazily wires the channel handler on first access; no enable/disable call. Cancel the `StreamSubscription` in `dispose()` (FLUTTER-BP-11). Not buffered for cold boot — by design, a tap can't precede Dart being up. |

### Diagnostics

| Method | `1.1.0` | Notes |
|---|---|---|
| `getInitialNotification() → Future<PushEngageResult<Map<String, dynamic>?>>` | ✅ | Recovers the notification that **cold-launched** the app (iOS, force-quit → tap) before Dart could subscribe to `deepLinkStream`. Idempotent drain: resolves `null` on Android and on every call after the first. Call once on startup (see `init-code.md`). |
| `runConfigValidation(String senderId, String projectId) → Future<PushEngageResult<bool>>` | ✅ Android only | Validates the device's FCM config (sender id / project id) against the dashboard. Always resolves `true` on iOS. Diagnostic only — not part of the permission flow. |

### Streams

| Member | `1.1.0` | Notes |
|---|---|---|
| `static Stream<Map<String, dynamic>?> get deepLinkStream` | ✅ | Broadcast stream. Lazy-initialized on first access — register listener in root widget's `initState` BEFORE any async startup work (FLUTTER-BP-02, KI-F003). Map shape: `{'deepLink': '<url-string>', ...}`; `data` is now always a `Map` on both platforms (Android no longer delivers a JSON string). **Cold-boot taps are NOT delivered here** — use `getInitialNotification()`. |
| `static Stream<FcmConfigError> get onFcmConfigError` | ✅ Android only | Emits `FcmConfigError` on an FCM sender-id/project-id mismatch or invalid local config. Never emits on iOS. |

## Model types

All model types are re-exported from the package root (`lib/pushengage_flutter_sdk.dart` is a barrel that exports every model file) — a single `import 'package:pushengage_flutter_sdk/pushengage_flutter_sdk.dart';` gives access to all of them. Importing a model file directly (e.g. `import 'package:pushengage_flutter_sdk/model/<name>.dart';`) also works but is unnecessary.

| Type | Constructor signature | Notes |
|---|---|---|
| `Environment` (enum) | `.staging`, `.production` | **NEW.** `wireValue` sends `'STAGING'` / `'PRODUCTION'` to native. Used by `setEnvironment`. |
| `IdentifyFields` | 12 optional named params — `firstName`, `lastName`, `email`, `phone`, `gender`, `dob`, `language`, `profileId`, `country`, `city`, `state`, `zip`; values `String`/`num`/`bool` | **NEW.** Implements `Mappable` (`toMap()` keys are snake_case). Used by `identify`. |
| `TrackEventPayload` | `TrackEventPayload({required String eventName, Map<String, Object>? data, String? profileId, String? provider, String? eventType})` | **NEW.** Implements `Mappable`; `toMap()` throws `ArgumentError` if `eventName` is empty. Used by `trackEvent`. |
| `FcmConfigError` | `const FcmConfigError({required int code, required String message})` | **NEW.** Codes 5001 (sender id mismatch), 5002 (project id mismatch), 5003 (local config invalid), 5004 (both sender id and project id mismatch). Emitted by `onFcmConfigError`. |
| `Goal` | `Goal({required String name, int? count, double? value})` | Named parameters; only `name` is required. |
| `TriggerCampaign` | Named parameters — `campaignName` and `eventName` required; `referenceId`, `profileId`, `data` optional | `data` is `Map<String, String>?` — values are `String`, NOT `dynamic`. Not positional. |
| `TriggerAlert(...)` | Named parameters | `expiryTimestamp` serializes as UTC ISO-8601 (millisecond precision + `Z`). |
| `DynamicSegment(...)` | Named parameters | Used by `addDynamicSegment`. |
| `TriggerStatusType` (enum) | `.enabled`, `.disabled` | Used by `automatedNotification`. |
| `IAMCustomAction` | `const IAMCustomAction({required String actionId, required Map<String, String> parameters})` | **NEW (1.1.0).** Emitted by `onIAMCustomAction`. **Route on `parameters['action']`** (the dashboard Action name); `actionId` is a dashboard-generated internal key (e.g. `t04B05`). `parameters` values are always `String` (stringified natively). Exported from the package barrel. |

TriggerCampaign, DynamicSegment, IdentifyFields, and TrackEventPayload implement the `Mappable` interface; Goal and TriggerAlert each expose a public toMap() but do NOT implement Mappable.

## Method-channel name

- `MethodChannel('PushEngage')` — single channel.
- Native method-name prefix: `PushEngage#<method>`. Useful for log filtering (e.g., `grep 'PushEngage#'` to see which Dart calls reached native).

## Minimum platform versions

| Layer | Minimum |
|---|---|
| Flutter SDK | `>=3.22.2` (raised in `1.1.0`; was `>=3.3.0`) |
| Dart | `>=3.4.3 <4.0.0` |
| iOS deployment target | `15.0` — the plugin podspec (`s.platform = :ios, '15.0'`) and the transitive `PushEngage`/`PushEngageExtension` `1.1.0` pods all require iOS 15.0. `flutter pub get` doesn't check it, so the constraint surfaces at `pod install` — set `platform :ios, '15.0'` in `ios/Podfile` and the `Runner` deployment target (KI-F013 / FLUTTER-AUDIT-15). The SDK example app uses `15.0`. |
| Android `minSdk` | `21` (the plugin's minimum). Native PushEngage Android SDK supports down to API 16, but the Flutter plugin caps at 21. |
| Android `compileSdk` | `34` |

## Companion library pins (Android side)

Use these EXACT versions. The Flutter SDK example app uses `google-services 4.4.0` and `firebase-bom 33.1.2`; for consistency with the native Android spoke (and to track current upstream), the recommended pins are:

| Library | Version | Coordinate |
|---|---|---|
| `google-services` Gradle plugin | `4.4.2` | `com.google.gms:google-services:4.4.2` |
| Firebase BOM | `33.5.1` | `platform('com.google.firebase:firebase-bom:33.5.1')` |
| Firebase Messaging | (from BOM) | `com.google.firebase:firebase-messaging` (no version — BOM provides it) |

## Stale public-doc version pins (use for audit findings)

Some earlier setup guides show these older values. Use this matrix as the source of truth.

| Public doc value | Actual current value | Audit ref |
|---|---|---|
| `pushengage_flutter_sdk: ^0.0.1` in pubspec | `^1.1.0` | FLUTTER-AUDIT-01 / KI-F001 |
| `pod 'PushEngage', '~>0.0.4'` in NSE Podfile snippet | `pod 'PushEngageExtension', '1.1.0'` | FLUTTER-AUDIT-03 / KI-F011 |
| `pod 'PushEngage', '0.0.4'` in NCE Podfile snippet | `pod 'PushEngageExtension', '1.1.0'` | FLUTTER-AUDIT-03 |
| NSE deployment target "iOS 10+" | iOS 15 (matches the pods' `15.0` floor from `1.1.0`) | — |

## Package channels

- **Flutter plugin:** pub.dev — `https://pub.dev/packages/pushengage_flutter_sdk`. Source of truth for the published version.
- **Source code:** `https://github.com/awesomemotive/pushengage-flutter-sdk`.
- **iOS pod:** transitively via the plugin's podspec → depends on the `PushEngage` pod (the iOS-native SDK app pod, same one the iOS spoke documents), which in turn depends transitively on `PushEngageExtension` — the extension-safe pod the NSE/NCE targets link directly.
- **Android library:** transitively via the plugin → depends on `com.github.awesomemotive:pushengage-android-sdk` (JitPack, same one the Android spoke documents).

## Known-issue version bounds

Maintained in `pushengage-debug/known-issues.md`. Flutter entries use IDs `KI-F001..KI-F014`. Each row references this matrix's `1.1.0` plugin baseline (with underlying native pins `PushEngage 1.1.0` + `PushEngageExtension 1.1.0` / `pushengage-android-sdk 1.0.0`).

---

## When the Flutter plugin version bumps

1. Bump the version in this file's "Current stable version" section.
2. Bump the `pubspec.yaml` snippet in `install.md`.
3. Re-verify the API surface table against the new public interface — read the published Dart API surface directly from the source repo.
4. Check if the transitive native pins changed (the plugin's iOS `PushEngage` pod dependency, the plugin's Android `implementation` dependency). If they bumped, sync this matrix's "Native dependencies" table.
5. Add or close known-issue rows in `pushengage-debug/known-issues.md` (KI-F00x entries).
6. Update `references.md` if the GitHub release tag URL changes.
7. Re-verify the IAM surface (`triggerIAMEvent`, `onIAMCustomAction`, `IAMCustomAction`) against `in-app-messaging.md` and the iOS deployment-target floor.
