# iOS Static Audit Checks

Audit runs first in the debugging flow (`pushengage-debug` Phase 1). Each check below:

- States what to read.
- States what pattern to look for.
- Cites the rule ID violated.
- Categorizes the finding as **Likely cause** / **Worth fixing** / **Cosmetic**.

Findings format (for output to the customer):

```
[Likely cause]  IOS-BP-08 — Manual swizzling without `registerDeviceToServer` forward
   File: AppDelegate.swift:42
   Effect: Device never registers with PushEngage; no notifications received.
   Fix: Add `application(_:didRegisterForRemoteNotificationsWithDeviceToken:)` and call
        PushEngage.registerDeviceToServer(with: deviceToken).
```

---

## Read targets

For each audit run, read (or glob the paths first):

- `**/AppDelegate.swift`, `**/AppDelegate.m`, `**/<App>App.swift` (SwiftUI)
- `**/Info.plist` (main app + any extension targets)
- `Podfile` (if present) — including any nested NSE/NCE extension-target blocks (check which pod each links: `PushEngage` vs `PushEngageExtension`)
- `Package.swift` and `*.xcodeproj/project.pbxproj` (for SPM packages + extension target presence)
- Any file matching `**/NotificationService.swift`, `**/NotificationViewController.swift`

Don't read further than this. The audit is bounded.

## Check IOS-AUDIT-01 — `setIntialInfo` typo (IOS-BP-06)

**Read:** all `.swift` and `.m` files (already loaded).
**Pattern:** `setIntialInfo` (note the missing 'i').
**Category:** Likely cause (compile error).
**Fix:** rename to `setInitialInfo`.

## Check IOS-AUDIT-02 — `enableLogs` instead of `enableLogging` (IOS-BP-07)

**Pattern:** `PushEngage.enableLogs` or `setEnableLogs:` or `setEnableLogs(`.
**Category:** Likely cause (compile error).
**Fix:** rename to `enableLogging` / `setEnableLogging:`.

## Check IOS-AUDIT-03 — Init order (IOS-BP-01)

**Read:** AppDelegate.
**Pattern:**
- `swizzleInjection` not inside `init()` → flag.
- `setNotificationOpenHandler` called before `setInitialInfo` in the same scope → flag.
- `setAppId` after `setInitialInfo` → flag.
**Category:** Worth fixing.
**Fix:** reorder.

## Check IOS-AUDIT-04 — Manual mode missing forwards (IOS-BP-08)

**Pattern:** if `swizzleInjection(isEnabled: false)` is present anywhere in AppDelegate, verify presence of all four delegate forwards:
- `didRegisterForRemoteNotificationsWithDeviceToken` + `PushEngage.registerDeviceToServer(with:`
- `didReceiveRemoteNotification:fetchCompletionHandler:` + `PushEngage.receivedRemoteNotification(`
- `willPresent` + `PushEngage.willPresentNotification(`
- `didReceive response` + `PushEngage.didReceiveRemoteNotification(with:`

**Category:** Likely cause (for each missing one) — silent breakage.
**Fix:** add the missing forward(s); see `init-code.md` manual template.

## Check IOS-AUDIT-05 — NCE without category fix (IOS-BP-09)

**Read:** if a Notification Content Extension target exists (detect via Info.plist with `NSExtensionPointIdentifier = com.apple.usernotifications.content-extension`), read the extension's Info.plist.
**Pattern:** `UNNotificationExtensionCategory` value is missing OR equals `myNotificationCategory`.
**Category:** Likely cause (extension never fires).
**Fix:** set to the actual category identifier sent by the dashboard.

## Check IOS-AUDIT-06 — App Group mismatch (IOS-BP-10)

**Read:** if NSE target exists, read main app Info.plist and NSE Info.plist.
**Pattern:**
- `PushEngage_App_Group_Key` missing from either → flag.
- Values don't match → flag.
- App Groups entitlement missing from either target → flag.
**Category:** Likely cause (NSE silently can't read main-app data).
**Fix:** match the group identifier on both targets; set `PushEngage_App_Group_Key` to that group in both Info.plists.

## Check IOS-AUDIT-07 — Missing post-install hook (IOS-BP-12) — conditional

**Read:** `Podfile` (if present).
**Pattern:** `pod 'PushEngage'` present BUT no `APPLICATION_EXTENSION_API_ONLY = 'NO'` post-install hook.
**Category:** Cosmetic — **no longer required for `1.0.0`.** The hook is only relevant if the `PushEngage` app pod is (incorrectly) linked into an extension target. With `1.0.0`'s two-module split, extension targets link only the extension-safe `PushEngageExtension` pod, so the hook is usually unnecessary. Do not flag a missing hook as a likely cause on `1.0.0`.
**Fix:** Only suggest adding the fallback hook from `install.md` if the build actually reports `APPLICATION_EXTENSION_API_ONLY` errors, or if you find the `PushEngage` pod linked into an extension target (in which case the real fix is IOS-AUDIT-15 — switch that target to `PushEngageExtension`).

## Check IOS-AUDIT-08 — Version pin loose (XP-BP-06)

**Read:** Podfile / Package.swift / `Package.resolved`.
**Pattern:** `PushEngage` OR `PushEngageExtension` dep using `~>`, `^`, or no version specified. (From `1.0.0` both pods exist; the extension target's `PushEngageExtension` dep must be pinned exactly too.)
**Category:** Worth fixing.
**Fix:** pin exactly to the version in `version-matrix.md` (`1.0.0` for both pods).

## Check IOS-AUDIT-09 — `enableLogging = true` unconditional (XP-BP-07)

**Pattern:** `PushEngage.enableLogging = true` or `setEnableLogging:YES` NOT inside `#if DEBUG` / `#ifdef DEBUG`.
**Category:** Worth fixing.
**Fix:** wrap in `#if DEBUG ... #endif`.

## Check IOS-AUDIT-10 — `requestNotificationPermission` at launch (XP-BP-03)

**Pattern:** `requestNotificationPermission` call inside `application(_:didFinishLaunchingWithOptions:)`.
**Category:** Cosmetic (works but bad UX).
**Fix:** defer behind a user action.

## Check IOS-AUDIT-11 — App ID UUID hardcoded (XP-BP-01)

**Pattern:** `PushEngage.setAppID(id: "<uuid-shaped-literal>")` where the literal matches `[0-9a-f]{8}-[0-9a-f]{4}-...`.
**Category:** Cosmetic (works but bad hygiene).
**Fix:** move to config / env-flag-aware build setting.

## Check IOS-AUDIT-13 — Foreground handler with long await (IOS-BP-05)

**Pattern:** inside the `setNotificationWillShowInForegroundHandler` closure, look for `await` over network or `asyncAfter` with `> 25` seconds.
**Category:** Cosmetic (may or may not be a real bug).
**Fix:** complete within 25 seconds or accept the auto-resolve.

## Check IOS-AUDIT-14 — `PushEngageAutoHandleDeeplinkURL` unset and no handler (IOS-BP-11)

**Pattern:** code references `PENotification.launchURL` or `notification.launchURL` somewhere, but Info.plist has no `PushEngageAutoHandleDeeplinkURL` key AND no `setNotificationOpenHandler` registered.
**Category:** Worth fixing (deep links will be silently dropped).
**Fix:** set the Info.plist key to YES, or register a handler.

## Check IOS-AUDIT-15 — Extension target imports wrong module/class (`1.0.0` two-module split)

**Read:** NSE/NCE source files (`**/NotificationService.swift`, `**/NotificationViewController.swift`) and the Podfile's nested extension-target blocks.
**Pattern:** in an extension (NSE/NCE) target, any of:
- `import PushEngage` (should be `import PushEngageExtension`), OR
- `PushEngage.didReceiveNotificationExtensionRequest` / `PushEngage.serviceExtensionTimeWillExpire` / `PushEngage.getCustomUIPayLoad` call sites (should use the `PushEngageExtension.` class prefix), OR
- the extension Podfile target using `pod 'PushEngage'` instead of `pod 'PushEngageExtension'` (SPM: linking the `PushEngage` product instead of `PushEngageExtension`).

**Category:** Likely cause (compile error — e.g. `Type 'PushEngage' has no member 'didReceiveNotificationExtensionRequest'`).
**Reference:** KI-006.
**Fix:** apply the three migration steps — dependency `pod 'PushEngage'` → `pod 'PushEngageExtension'` (or SPM product), `import PushEngage` → `import PushEngageExtension`, and class prefix `PushEngage.` → `PushEngageExtension.` for the three NSE/NCE methods. See `nse-setup.md` / `nce-setup.md`.

---

## Output formatting (when invoked from debug flow)

Group findings by category:

```
**Likely cause** (X findings):
1. [IOS-BP-08] ...
2. [IOS-BP-09] ...

**Worth fixing** (X findings):
1. [IOS-BP-01] ...

**Cosmetic** (X findings):
1. [XP-BP-03] ...
```

If 0 findings: report "No iOS audit issues found." and hand off to debug Phase 2 (known-issues match).
