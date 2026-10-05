# iOS SDK Version Matrix

Single source of truth for iOS-SDK-version-pinned data. Other files in this spoke reference this one.

## Current stable version

**iOS SDK:** `1.1.0` — pinned exactly in `install.md`, `nse-setup.md`, `nce-setup.md`, and the audit advisory.

> **Release note (2026-09-30):** `1.1.0` adds **In-App Messaging** (`triggerIAMEvent`, `setIAMCustomActionHandler`, `PEIAMCustomActionHandler`) inside the existing `PushEngage` app module — no new pod/product, no init change. **Breaking:** the minimum deployment target rises from iOS 12.0 to **iOS 15.0** (both pods; "the lowest version Xcode 27 builds for"). Apps that must keep iOS 12–14 stay on `1.0.0` and do not get IAM. Optional opt-in: background campaign refresh via `BGTaskSchedulerPermittedIdentifiers` = `com.pushengage.iam.refresh` + Background fetch mode. See `in-app-messaging.md`.
>
> **Structural note (2026-06):** `1.0.0` split the SDK into TWO modules/pods: `PushEngage` (app target) and `PushEngageExtension` (NSE/NCE targets). App-target code is unchanged from the app-side `0.1.0` surface; extension targets must use the `PushEngageExtension` pod/product, `import PushEngageExtension`, and the `PushEngageExtension.` call prefix. This split is unchanged in `1.1.0`.

## Public API surface (per version)

### App-target API (module `PushEngage`)

| Method | `1.1.0` | Notes |
|---|---|---|
| `setAppID(id:)` | ✅ | Init order: after `swizzleInjection`, before optional handlers. Obj-C: `setAppIDWithId:`. `setAppId(key:)` is an internal method, not the public API — use `setAppID(id:)`. |
| `setInitialInfo(for:with:)` | ✅ | Public doc snippet contains typo `setIntialInfo` — flag in audit. |
| `setEnvironment(environment:)` | ✅ | `PEEnvironment`: `.staging`, `.production`. Optional; production is the default. If used, call **before** `setAppID` so the SDK picks the right base URLs. |
| `swizzleInjection(isEnabled:)` | ✅ | Called in `init()` only. |
| `enableLogging` (property) | ✅ | The property is `enableLogging`, **not `enableLogs`**. From `1.0.0` the flag is mirrored to the shared app-group container for the current session, so NSE/NCE processes pick it up automatically (no separate extension call) — it is not a durable cross-launch preference. |
| `setNotificationOpenHandler(block:)` | ✅ | Fires on notification tap/open; no timeout. |
| `setNotificationWillShowInForegroundHandler(block:)` | ✅ | Spelled `Foreground` (watch the spelling). 25s auto-resolve timeout (IOS-BP-05). |
| `requestNotificationPermission(completion:)` | ✅ | Async grant/deny callback. |
| `registerDeviceToServer(with:)` | ✅ | Required in manual swizzling-off mode. |
| `receivedRemoteNotification(application:userInfo:completionHandler:)` | ✅ | Required in manual swizzling-off mode. It is `@discardableResult` and returns `Bool`; the `completionHandler` param is optional (same signature as `0.1.0`). Existing void call sites compile. |
| `willPresentNotification(center:notification:completionHandler:)` | ✅ | Required in manual swizzling-off mode. |
| `didReceiveRemoteNotification(with:)` | ✅ | Required in manual swizzling-off mode. |
| `getSdkVersion()` | ✅ | Returns `"1.1.0"`. Added at iOS SDK `0.1.0`; present in this pinned baseline. |
| `setBadgeCount(count:)` | ✅ | |
| `setPlatform(_:)` | ✅ | Wrapper User-Agent flavor; called by the RN/Flutter wrappers. Native iOS apps normally don't call it. |
| `setWrapperVersion(_:)` | ✅ | Wrapper User-Agent version; wrapper-only (see `setPlatform`). |
| `trackEvent(name:properties:profileId:provider:eventType:completionHandler:)` | ✅ | |
| `identify(fields:completionHandler:)` | ✅ | |
| `logout(fieldNames:completionHandler:)` | ✅ | |
| `getNotificationPermissionStatus()` | ✅ | Returns `"granted"` / `"denied"` / `"notYetRequested"`. |
| `getSubscriptionStatus(completionHandler:)` | ✅ | |
| `getSubscriptionNotificationStatus(completionHandler:)` | ✅ | |
| `add(attributes:completionHandler:)` | ✅ | Deprecated in `1.0.0` — renamed `addSubscriberAttributes`. Still compiles (with warning). |
| `set(attributes:completionHandler:)` | ✅ | Deprecated in `1.0.0` — renamed `setSubscriberAttributes`. Still compiles (with warning). |
| `addSubscriberAttributes(_:completionHandler:)` | ✅ | Alias of `add(attributes:)`. |
| `setSubscriberAttributes(_:completionHandler:)` | ✅ | Alias of `set(attributes:)`. |
| `getSubscriberAttributes(completionHandler:)` | ✅ | |
| `addProfile(for:completionHandler:)` | ✅ | |
| `deleteSubscriberAttributes(for:completionHandler:)` | ✅ | |
| `addSegments(_:completionHandler:)` | ✅ | |
| `removeSegments(_:completionHandler:)` | ✅ | |
| `addDynamicSegments(_:completionHandler:)` | ✅ | |
| `automatedNotification(status:completionHandler:)` | ✅ | `TriggerStatusType`: `.enabled`, `.disabled`. |
| `subscribe(completionHandler:)` | ✅ | |
| `unsubscribe(completionHandler:)` | ✅ | |
| `sendGoal(goal:completionHandler:)` | ✅ | `Goal(name:count:value:)`. |
| `sendTriggerEvent(triggerCampaign:completionHandler:)` | ✅ | |
| `addAlert(triggerAlert:completionHandler:)` | ✅ | |
| `getSubscriberId(completion:)` | ✅ | |
| `getSubscriberDetails(for:completionHandler:)` | ✅ | |
| `triggerIAMEvent(eventName:parameters:completionHandler:)` | ✅ NEW (1.1.0) | In-App Messaging custom trigger. `parameters: [String: Any]? = nil`; `completionHandler: ((Bool, Error?) -> Void)?` has NO default — pass `nil` explicitly if unused. Obj-C: `triggerIAMEventWithEventName:parameters:completionHandler:`. Empty `eventName` → `(false, "Event name is required")`. `true` = event accepted, NOT "a message was shown". Works without a push subscription. See `in-app-messaging.md`. |
| `setIAMCustomActionHandler(_:)` | ✅ NEW (1.1.0) | Registers the `PEIAMCustomActionHandler` closure `(actionId: String, parameters: [String: String]) -> Void` for `custom` IAM buttons. **Route on `parameters["action"]`** (the dashboard Action name); `actionId` is a dashboard-generated internal key (e.g. `"t04B05"`). **Single slot** — a second call replaces the first; no getter, no removal (IOS-BP-16). Obj-C: `setIAMCustomActionHandler:`. |
| `PEIAMCustomActionHandler` (typealias) | ✅ NEW (1.1.0) | `(_ actionId: String, _ parameters: [String: String]) -> Void`. The only public IAM type; every other IAM type (`IAMController`, `IAMPosition`, …) is `internal` — never reference them. |

### Notification Extension API (module `PushEngageExtension`)

These three methods **moved off the `PushEngage` class** in `1.0.0`. In extension (NSE/NCE) targets, `import PushEngageExtension` and call them on **`PushEngageExtension`**, NOT `PushEngage`. Calling them on `PushEngage` no longer compiles (`Type 'PushEngage' has no member …`).

| Method | `1.1.0` | Notes |
|---|---|---|
| `PushEngageExtension.didReceiveNotificationExtensionRequest(_:bestContentHandler:)` | ✅ | NSE use only. Same signature/name as before; moved from `PushEngage`. |
| `PushEngageExtension.serviceExtensionTimeWillExpire(_:content:)` | ✅ | NSE use only. Returns `UNMutableNotificationContent?`. Moved from `PushEngage`. |
| `PushEngageExtension.getCustomUIPayLoad(for:)` | ✅ | NCE use only. Returns `CustomUIModel` (`title`, `body`, `image: UIImage?`, `buttons: [CustomUIButtons]?`). Moved from `PushEngage`. |

## Min deployment target

iOS `15.0+` (raised from `12.0` in `1.1.0` — breaking). SPM additionally requires Xcode 15+ / Swift 5.9. If the customer's deployment target is below 15.0, `pod install` fails with a platform-compatibility error (KI-009); either raise the target to 15.0 or stay on `1.0.0` (no IAM).

## Package channels

- SPM: `https://github.com/awesomemotive/pushengage-ios-sdk` — tag `1.1.0`. Two products: link `PushEngage` to the app target, `PushEngageExtension` to NSE/NCE targets. IAM lives in `PushEngage` only.
- CocoaPods: app target → `pod 'PushEngage', '1.1.0'`; NSE/NCE target(s) → `pod 'PushEngageExtension', '1.1.0'`. `PushEngage` transitively depends on `PushEngageExtension` at the same version. From `1.1.0` the `PushEngage` podspec ships a `PushEngageResources` resource bundle (the IAM Core Data model); CocoaPods and SPM copy it automatically — only manual/static-framework integrations must copy `PushEngageResources.bundle` themselves, otherwise IAM logs `CoreData model not found in resource bundle` and disables itself.

## Known-issue version bounds

Maintained in `pushengage-debug/known-issues.md`. Each entry has `affected: ≥A.B[, <C.D]` and `fixed_in:` columns referencing this matrix.

---

When the SDK version bumps:
1. Bump the version in this file.
2. Bump the version pin in `install.md`.
3. Re-verify the API surface table against the new public interface.
4. Add or close known-issue rows in `pushengage-debug/known-issues.md`.
5. Re-verify the two-module split is reflected in `install.md`, `nse-setup.md`, and `nce-setup.md` (app target → `PushEngage`; NSE/NCE → `PushEngageExtension`).
6. `IOS-AUDIT-12` (the `getSdkVersion()` not-in-pinned-SDK check) was retired when the baseline moved to ≥ `0.1.0`; it no longer applies at `1.1.0`. Re-add a similar check only if a future API is added that the pinned baseline lacks.
7. Re-verify the IAM surface (`triggerIAMEvent`, `setIAMCustomActionHandler`) and the min-deployment-target row against `in-app-messaging.md` and the audit checks IOS-AUDIT-16..18.
