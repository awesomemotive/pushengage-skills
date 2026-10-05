# Android SDK Version Matrix

Single source of truth for Android-SDK-version-pinned data. Other files in this spoke reference this one.

## Current stable version

**Android SDK:** `1.0.0` — pinned exactly in `install.md` and the audit advisory.

> **Release note (2026-09-29):** Android SDK `1.0.0` (JitPack tag `1.0.0`; `getSdkVersion()` returns `"1.0.0"`) adds **In-App Messaging**: `triggerIAMEvent(...)` (3 overloads), `setIAMCustomActionHandler(IAMCustomActionHandler)`, and the `com.pushengage.pushengage.iam.action.IAMCustomActionHandler` interface. No new Gradle dependency, host manifest entry, permission, minSdk change, or init change — IAM activates inside `Builder.build()`; the library manifest merges in an internal `<provider>` automatically. The AAR now ships consumer R8/ProGuard rules, so minified builds need no host configuration. Room schema migrates 1→2 automatically. `requestNotificationPermission` still requires `ComponentActivity`. See `in-app-messaging.md`.
>
> **Release note (2026-06):** Android SDK `0.1.0` (released 2026-06-05, JitPack tag `0.1.0`, `versionCode 7`; `getSdkVersion()` returns `"0.1.0"`). Adds the User Identification API (`identify`, `logout`, `trackEvent`), `setBadgeCount`, and `setFcmConfigErrorListener`. Additive and non-breaking from `0.0.6`. No dependency-pin, minSdk, or install-step change.

## Public API surface (per version)

| Method | `1.0.0` | Notes |
|---|---|---|
| `PushEngage.Builder().addContext(Context).setAppId(String).build()` | ✅ | Builder pattern. Called in `Application.onCreate()`. |
| `requestNotificationPermission(ComponentActivity, PushEngagePermissionCallback)` | ✅ | **`ComponentActivity` required.** Plain `Activity` does NOT compile. `AppCompatActivity` / `FragmentActivity` / `androidx.activity.ComponentActivity` all qualify. (See ANDROID-BP-01 / KI-A002.) |
| `getNotificationPermissionStatus()` → `String` | ✅ | Returns only `"granted"` / `"denied"`. No "not_determined" third state. |
| `subscribe()` | ✅ | Auto-called on permission grant; rarely needs manual invocation. |
| `setSmallIconResource(String)` | ✅ | Pass a drawable **name** that exists in `res/drawable*/`; if it doesn't resolve, the SDK falls back to its bundled default bell icon (`ic_stat_notification_default`). (See ANDROID-BP-04.) |
| `enableLogging(Boolean)` | ✅ | Same name across all platforms. Gate behind `BuildConfig.DEBUG`. |
| `getSdkVersion()` → `String` | ✅ | **Available on Android.** Returns the SDK version string (now `"1.0.0"`). Always present on Android; iOS lacked it through `0.0.6` and added it in `0.1.0` (the iOS spoke now pins `1.1.0`, which has it). |
| `getSubscriberId(PushEngageResponseCallback)` | ✅ | Callback-based. `PushEngageResponseCallback` has TWO abstract methods — `onSuccess(Object)` and `onFailure(Integer, String)` — so Kotlin lambda/SAM conversion does NOT compile; use an `object :` expression. Treat the result as PII (XP-BP-05). |
| `getDeviceTokenHash()` | ⚠️ | Returns the stored device-token hash synchronously. Not deprecated (no `@Deprecated` in source), but prefer `getSubscriberId` in new integrations — it is async and the cross-platform-consistent API. Note it is not identical to `getSubscriberId`: this returns the stored hash unconditionally, whereas `getSubscriberId` returns null unless subscribed. |
| `sendTriggerEvent(trigger)` | ✅ | Parameter is named `trigger` — NOT `triggerCampaign`. |
| `addAlert(...)` | ✅ | Uses nested enums `PushEngage.TriggerAlertType.X` and `PushEngage.TriggerAlertAvailabilityType.X` (qualified through outer class). |
| `identify(JSONObject)` / `identify(JSONObject, PushEngageResponseCallback)` | ✅ NEW | User Identification. Valid keys = 12 predefined fields (`first_name,last_name,email,phone,gender,dob,language,profile_id,country,city,state,zip`). Values String/Number/Boolean only. Numeric `profile_id` coerced to String. Cache short-circuit (24h TTL). |
| `logout(List<String>)` / `logout(List<String>, PushEngageResponseCallback)` | ✅ NEW | Removes fields. `null`/empty → PII default `[first_name,last_name,email,phone,gender,dob,profile_id]`. |
| `trackEvent(TrackEvent)` / `trackEvent(TrackEvent, PushEngageResponseCallback)` | ✅ NEW | `TrackEvent` HAS `@JvmOverloads` (only model class that does). `provider` defaults `"PushEngage"`, `eventType` defaults `"PushEngage.CustomEvent"`. |
| `setBadgeCount(int)` | ✅ NEW | `0` → cancels all notifications (clears launcher dot); `>0` → `setNumber()` on future SDK notifications; negatives → 0. No system numeric launcher badge on Android. |
| `setFcmConfigErrorListener(FcmConfigErrorListener)` | ✅ NEW | Fires `PEErrorCodes` 5001/5002/5003/5004 on FCM config mismatch (sender-id / project-id / local-config-invalid / both). `FcmConfigErrorListener` is a Kotlin `fun interface` (SAM/lambda OK). `null` clears. |
| `setWrapperVersion(String)` | ✅ NEW (wrapper-only) | For Flutter/RN wrapper plugins; NOT a host-app API. Empty/null clears. |
| `getSubscriptionStatus(PushEngageResponseCallback)` | ✅ (since 0.0.6) | `onSuccess(Object)` carries a `Boolean`. |
| `getSubscriptionNotificationStatus(PushEngageResponseCallback)` | ✅ (since 0.0.6) | `Boolean`: subscribed AND permission granted. |
| `unsubscribe(PushEngageResponseCallback)` | ✅ (since 0.0.6) | Server status → unsubscribed; reverts local flag on API failure. |
| `subscribe(ComponentActivity, PushEngageResponseCallback)` | ✅ (since 0.0.6) | Manual subscribe; requests permission if not granted. Requires `ComponentActivity`. |
| `addProfileId(String)` / `addProfileId(String, PushEngageResponseCallback)` | ✅ (since 0.0.6) | Also merges `profile_id` into the local subscriber-fields cache. |
| `addSegment(List<String>[, callback])` / `removeSegment(List<String>[, callback])` | ✅ (since 0.0.6) | |
| `addDynamicSegment(List<AddDynamicSegmentRequest.Segment>[, callback])` | ✅ (since 0.0.6) | |
| `addSubscriberAttributes(JSONObject[, callback])` / `setSubscriberAttributes(JSONObject[, callback])` | ✅ (since 0.0.6) | `org.json.JSONObject`. `set…` replaces all attributes. Use these (not `identify`) for arbitrary custom keys. |
| `deleteSubscriberAttributes(List<String>[, callback])` / `getSubscriberAttributes(PushEngageResponseCallback)` / `getSubscriberDetails(List<String>, PushEngageResponseCallback)` | ✅ (since 0.0.6) | |
| `sendGoal(Goal[, callback])` | ✅ (since 0.0.6) | |
| `automatedNotification(TriggerStatusType[, callback])` | ✅ (since 0.0.6) | |
| `triggerIAMEvent(String eventName)` / `triggerIAMEvent(String, Map<String, Object> parameters)` / `triggerIAMEvent(String, Map<String, Object>, PushEngageResponseCallback)` | ✅ NEW (1.0.0) | In-App Messaging custom trigger. Blank name → `onFailure(400, "Event name is required")`. Deliberately NOT gated on subscription/permission/site status. `onSuccess(null)` = event accepted, NOT "a message was shown". Parameter values are stringified before matching (`120`, `120.0` → `"120"`). See `in-app-messaging.md`. |
| `setIAMCustomActionHandler(IAMCustomActionHandler handler)` | ✅ NEW (1.0.0) | Receives `custom` IAM button taps. `null` clears. Safe to call BEFORE `Builder.build()` (stashed and applied when IAM is ready). Register app-wide in `Application.onCreate()` (ANDROID-BP-16). Callback runs on the UI thread. |
| `IAMCustomActionHandler` (interface, `com.pushengage.pushengage.iam.action`) | ✅ NEW (1.0.0) | `fun onCustomAction(actionId: String, parameters: Map<String, Any>)`. **Route on `parameters["action"]`** (dashboard Action name); `actionId` is a dashboard-generated internal key (e.g. `"t04B05"`). Kotlin `interface` (NOT `fun interface`) → Kotlin needs `object : IAMCustomActionHandler { … }`; Java lambdas work. Scalar parameter values arrive as `String`s. The ONLY public IAM type — `IAMController`, `IAMConfigurationManager`, `IAMPosition`, etc. are `internal` / `@RestrictTo` and must never appear in customer code (ANDROID-BP-17). |

### Data class shapes (Kotlin)

| Type | Shape | Notes |
|---|---|---|
| `Goal(name, count, value)` | `count: Int?`, `value: Double?` (both NULLABLE) | No `@JvmOverloads`. |
| `TriggerCampaign(campaignName, eventName, referenceId?, profileId?, data?)` | 2 required + 3 Kotlin-default optional | Kotlin callers can pass 2 args; Java callers pass all 5 positional args (use `null` for optionals). See ANDROID-BP-09. |
| `TriggerAlert(type, productId, link, price, variantId?, expiryTimestamp?, alertPrice?, availability?, profileId?, mrp?, data?)` | 4 required + 7 Kotlin-default optional | `expiryTimestamp` is `Date`, NOT `String`. Java needs all 11 (no `@JvmOverloads`). |
| `TrackEvent(eventName, data?, profileId?, provider?, eventType?)` | 1 required + 4 optional | **HAS `@JvmOverloads`** → Java may pass 1-5 args. |

## Dependency coordinates

- Maven coordinate (JitPack-hosted, NOT Maven Central):
  `com.github.awesomemotive:pushengage-android-sdk:1.0.0`
- Required repository: `maven { url 'https://jitpack.io' }` (Groovy) / `maven { url = uri("https://jitpack.io") }` (Kotlin DSL).
- Add JitPack inside `dependencyResolutionManagement { repositories { ... } }` in `settings.gradle*` for modern projects, or inside `allprojects { repositories { ... } }` in the project-level `build.gradle*` for legacy.

## Companion library pins

Recommended host-app pins for new integrations. Note: the SDK itself builds and tests against `firebase-bom:26.1.1` / `google-services:4.3.15` — those older versions work, but modern apps targeting SDK 34 should use the newer pins below; Firebase BOMs are backwards compatible.

| Library | Version | Coordinate |
|---|---|---|
| `google-services` Gradle plugin | `4.4.2` | `com.google.gms:google-services:4.4.2` (project-level classpath) |
| Firebase BOM | `33.5.1` | `platform('com.google.firebase:firebase-bom:33.5.1')` |
| Firebase Messaging | (from BOM) | `com.google.firebase:firebase-messaging` (no version — BOM provides it) |

## SDK level guidance

| Level | Value | Rationale |
|---|---|---|
| `minSdk` (supported) | `16` | SDK's actual minimum. |
| `minSdk` (recommended) | `21` | Covers >99% of active devices; default for new integrations. Public doc states `21` as the minimum — that's actually the recommended floor; SDK supports down to `16` if a customer is constrained. |
| `targetSdk` | `34` | Required for Play Store submissions; aligns with current Android baseline. |
| `compileSdk` | `34` | Match `targetSdk`. |

If a customer is constrained to `minSdk 16..20`, the SDK works — note that the runtime notification permission flow (`POST_NOTIFICATIONS`) only triggers on API 33+.

## Package channels

- JitPack (only): `com.github.awesomemotive:pushengage-android-sdk:1.0.0` (IAM ships from this version; `0.1.0` and earlier have no IAM API).
- Source of truth for tags: `https://github.com/awesomemotive/pushengage-android-sdk` (see `references.md`).
- NOT published to Maven Central or Google's Maven repo. Do NOT suggest `mavenCentral()` as a sole repository — JitPack must be present.

## Known-issue version bounds

Maintained in `pushengage-debug/known-issues.md`. Android entries use IDs in the `KI-A0xx` range. Each row references this matrix's `1.0.0` baseline. For `Goal`/`TriggerCampaign`/`TriggerAlert`, Java callers pass all positional args (Kotlin can use defaults); `TrackEvent` (0.1.0) accepts 1–5 args.

---

## When the SDK version bumps

1. Bump the version in this file (current-stable header and the API surface table).
2. Bump the version pin in `install.md`.
3. Re-verify the API surface table against the new public interface — pay special attention to:
   - Whether `requestNotificationPermission` signature relaxes the `ComponentActivity` requirement.
   - Whether `setSmallIconResource` gains a log warning on missing drawable.
   - Whether Kotlin data classes (`TriggerCampaign`, `TriggerAlert`, `Goal`) gain `@JvmOverloads`.

   As of `1.0.0` (verified): `requestNotificationPermission` still requires `ComponentActivity`; `setSmallIconResource` still falls back silently with no missing-drawable log; and only the new `TrackEvent` class gained `@JvmOverloads` — `Goal`, `TriggerCampaign`, and `TriggerAlert` still lack it.
4. Re-check companion library pins. Bump `google-services` and `firebase-bom` only after testing — Firebase BOM bumps occasionally break transitive deps.
5. Add or close known-issue rows in `pushengage-debug/known-issues.md` (KI-A00x entries).
6. Update `references.md` if the GitHub release tag URL changes.
7. Re-verify the IAM surface (`triggerIAMEvent` overloads, `setIAMCustomActionHandler`, `IAMCustomActionHandler`) against `in-app-messaging.md` and audit checks ANDROID-AUDIT-13/14.
