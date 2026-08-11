# iOS SDK Version Matrix

Single source of truth for iOS-SDK-version-pinned data. Other files in this spoke reference this one.

## Current stable version

**iOS SDK:** `1.0.0` — pinned exactly in `install.md`, `nse-setup.md`, `nce-setup.md`, and the audit advisory.

> **Structural note (2026-06):** `1.0.0` splits the SDK into TWO modules/pods: `PushEngage` (app target) and `PushEngageExtension` (NSE/NCE targets). App-target code is unchanged from the app-side `0.1.0` surface; extension targets must use the `PushEngageExtension` pod/product, `import PushEngageExtension`, and the `PushEngageExtension.` call prefix. Min deployment target is iOS 12.0.

## Public API surface (per version)

### App-target API (module `PushEngage`)

| Method | `1.0.0` | Notes |
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
| `getSdkVersion()` | ✅ | Returns `"1.0.0"`. Added at iOS SDK `0.1.0`; present in this pinned baseline. |
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

### Notification Extension API (module `PushEngageExtension`)

These three methods **moved off the `PushEngage` class** in `1.0.0`. In extension (NSE/NCE) targets, `import PushEngageExtension` and call them on **`PushEngageExtension`**, NOT `PushEngage`. Calling them on `PushEngage` no longer compiles (`Type 'PushEngage' has no member …`).

| Method | `1.0.0` | Notes |
|---|---|---|
| `PushEngageExtension.didReceiveNotificationExtensionRequest(_:bestContentHandler:)` | ✅ | NSE use only. Same signature/name as before; moved from `PushEngage`. |
| `PushEngageExtension.serviceExtensionTimeWillExpire(_:content:)` | ✅ | NSE use only. Returns `UNMutableNotificationContent?`. Moved from `PushEngage`. |
| `PushEngageExtension.getCustomUIPayLoad(for:)` | ✅ | NCE use only. Returns `CustomUIModel` (`title`, `body`, `image: UIImage?`, `buttons: [CustomUIButtons]?`). Moved from `PushEngage`. |

## Min deployment target

iOS `12.0+`. SPM additionally requires Xcode 15+ / Swift 5.9.

## Package channels

- SPM: `https://github.com/awesomemotive/pushengage-ios-sdk` — tag `1.0.0`. Two products: link `PushEngage` to the app target, `PushEngageExtension` to NSE/NCE targets.
- CocoaPods: app target → `pod 'PushEngage', '1.0.0'`; NSE/NCE target(s) → `pod 'PushEngageExtension', '1.0.0'`. `PushEngage` transitively depends on `PushEngageExtension` at the same version.

## Known-issue version bounds

Maintained in `pushengage-debug/known-issues.md`. Each entry has `affected: ≥A.B[, <C.D]` and `fixed_in:` columns referencing this matrix.

---

When the SDK version bumps:
1. Bump the version in this file.
2. Bump the version pin in `install.md`.
3. Re-verify the API surface table against the new public interface.
4. Add or close known-issue rows in `pushengage-debug/known-issues.md`.
5. Re-verify the two-module split is reflected in `install.md`, `nse-setup.md`, and `nce-setup.md` (app target → `PushEngage`; NSE/NCE → `PushEngageExtension`).
6. `IOS-AUDIT-12` (the `getSdkVersion()` not-in-pinned-SDK check) was retired when the baseline moved to ≥ `0.1.0`; it no longer applies at `1.0.0`. Re-add a similar check only if a future API is added that the pinned baseline lacks.
