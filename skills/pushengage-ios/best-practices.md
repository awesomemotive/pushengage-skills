# iOS PushEngage Best Practices (IOS-BP)

iOS-specific rules. Cite by rule ID in audit findings and debugging output. Cross-platform rules live in `pushengage/best-practices.md` (`XP-BP-NN`).

---

## IOS-BP-01 — Init order matters

**Rule:** Call PushEngage methods in this order — `swizzleInjection(isEnabled:)` in `AppDelegate.init()` only; then in `application(_:didFinishLaunchingWithOptions:)`, in this sequence: `setAppID(id:)` → `setInitialInfo(for:with:)` → optional handlers (`setNotificationOpenHandler`, `setNotificationWillShowInForegroundHandler`) → `enableLogging = true` (debug only).

**Why:** `swizzleInjection` must run before `UIApplication` finishes initializing the delegate, which is why it lives in `init()`. `setInitialInfo` reads from `launchOptions` and must run before any handler can be invoked, because the handler may need to know about a cold-boot launch. Handlers registered after `setInitialInfo` benefit from the SDK's replay queue.

**How to apply:** Audit check — read `AppDelegate.swift` (or SwiftUI app's adapter). Flag any of:
- `swizzleInjection` outside `init()`.
- `setAppID` after `setInitialInfo`.
- `setNotificationOpenHandler` before `setInitialInfo` (handler will register against an unprimed SDK).

## IOS-BP-02 — Prefer swizzling unless a hard constraint forces manual

**Rule:** Use `swizzleInjection(isEnabled: true)`. Only switch to manual when:
- Another SDK already swizzles `UIApplicationDelegate` AND its docs warn against double-swizzling (Firebase Messaging with `FirebaseAppDelegateProxyEnabled = YES` is the canonical case).
- The customer's policy disallows runtime swizzling (compliance, audit).
- The app uses an unusual delegate setup (multiple AppDelegates) where swizzling is documented to misbehave.

**Why:** Manual mode requires four delegate forwards (see IOS-BP-08) and is easier to get subtly wrong than swizzling. Most apps don't need manual mode.

**How to apply:** Default templates in `init-code.md` use swizzling. Manual templates exist but are gated behind one of the conditions above.

## IOS-BP-03 — Declare `remote-notification` background mode only when silent pushes are used

**Rule:** Don't add "Remote notifications" to Background Modes capability if you're only using user-visible (alert) notifications.

**Why:** App Store reviewers occasionally flag apps that declare the capability without justifying use. Silent pushes are also rate-limited by Apple, so declaring the capability you don't use is pure cost.

**How to apply:** When auditing manually — if `UIBackgroundModes` contains `remote-notification` but the codebase has no `content-available: 1` handler logic (look for `notification.contentAvailable == 1` or similar), flag as worth reconsidering. (No numbered IOS-AUDIT entry; judgment call.)

## IOS-BP-04 — Pin `.p12` private key password separately from source

**Rule:** Don't commit the `.p12` private key password to source control or paste it into any chat / ticket.

**Why:** With the `.p12` file and the password, anyone can send push notifications as your app. Apple revokes the cert if it leaks publicly.

**How to apply:** Inform customers of this verbally during the dashboard setup step. No audit check (the password isn't in customer code by default — only flag if seen).

## IOS-BP-05 — Foreground handler must complete within 25 seconds

**Rule:** `setNotificationWillShowInForegroundHandler`'s completion block must be called within 25 seconds. Don't await long async work inside it. (The public method is spelled `Foreground` — watch the spelling.)

**Why:** The SDK auto-resolves after 25s, which may not be what the customer wants (the notification might be shown when they intended to silence it, or vice versa).

**How to apply:** Audit check — if the handler block contains `await` over a network call or `DispatchQueue.global().asyncAfter(deadline: .now() + N)` with `N > 25`, flag it.

## IOS-BP-06 — `setIntialInfo` is a typo

**Rule:** The method is `setInitialInfo(for:with:)` — watch the spelling. A common typo is `setIntialInfo` (missing an 'i'), which fails with "Type 'PushEngage' has no member 'setIntialInfo'."

**Why:** This is a real, recurring source of "fresh integration doesn't link" support tickets.

**How to apply:** Audit check — grep `setIntialInfo` anywhere in `.swift` files. If present, flag with this rule ID and offer to fix.

## IOS-BP-07 — `enableLogging`, not `enableLogs`

**Rule:** The SDK property is `enableLogging` (Swift) / `setEnableLogging:` (Obj-C). A common mistake is `enableLogs` / `setEnableLogs:`.

**Why:** Same as IOS-BP-06 — recurring copy-paste failure.

**How to apply:** Audit check — grep `PushEngage.enableLogs\b` or `setEnableLogs:` in customer code. If present, flag and offer to rewrite.

## IOS-BP-08 — Manual swizzling mode requires four delegate forwards

**Rule:** When `swizzleInjection(isEnabled: false)`, the customer must forward ALL FOUR of these:

| Delegate method | PushEngage call | If missing |
|---|---|---|
| `didRegisterForRemoteNotificationsWithDeviceToken` | `PushEngage.registerDeviceToServer(with:)` | Device never registered → no notifications received |
| `didReceiveRemoteNotification:fetchCompletionHandler:` | `PushEngage.receivedRemoteNotification(application:userInfo:completionHandler:)` | Background/silent pushes not processed |
| `userNotificationCenter(_:willPresent:withCompletionHandler:)` | `PushEngage.willPresentNotification(center:notification:completionHandler:)` | Foreground presentation broken |
| `userNotificationCenter(_:didReceive:withCompletionHandler:)` | `PushEngage.didReceiveRemoteNotification(with:)` | Click/open analytics + action-button taps untracked |

**Why:** Each is documented in the SDK as "should only be implemented if you have disabled method swizzling." Missing one breaks integration in non-obvious ways.

**How to apply:** Audit check — if `swizzleInjection(isEnabled: false)` is present, verify all four forwards exist. Flag each missing one with its specific breakage.

## IOS-BP-09 — NSE Info.plist needs the right category for NCE to fire

**Rule:** If the customer adds a Notification Content Extension, the extension's Info.plist must set `NSExtension → NSExtensionAttributes → UNNotificationExtensionCategory` to the category identifier the NCE code checks (and that the dashboard sends as `aps.category`). Xcode's default template leaves this as `myNotificationCategory`.

**Why:** iOS only routes notifications to the NCE if `aps.category` matches a value in the extension's `UNNotificationExtensionCategory` array. Mismatch = extension never fires, custom UI never shows.

**How to apply:** Audit check — if NCE target exists, read its Info.plist. If `UNNotificationExtensionCategory` is missing or still `myNotificationCategory`, flag it.

## IOS-BP-10 — App Groups must match between main app and NSE

**Rule:** When NSE is added, both the main app and the NSE target must have the App Groups capability with the SAME group identifier, AND both Info.plists must contain `PushEngage_App_Group_Key` with that same group identifier as the value.

**Why:** PushEngage uses the App Group's shared `UserDefaults` to ferry data between the main app and the NSE process. Mismatch = NSE can't read what the main app wrote.

**How to apply:** Audit check — if NSE target exists, verify both Info.plists have `PushEngage_App_Group_Key`, both targets have the App Groups entitlement, and the group identifiers match.

## IOS-BP-11 — `PushEngageAutoHandleDeeplinkURL` defaults to NO

**Rule:** Without `PushEngageAutoHandleDeeplinkURL = YES` in Info.plist, http URLs in notification `pe.u` are NOT auto-opened — they're handed to the customer's `setNotificationOpenHandler` for the customer to act on.

**Why:** Customers often expect "notifications with a URL should open Safari." If they don't set the flag AND don't implement `setNotificationOpenHandler`, the URL is dropped silently.

**How to apply:** Audit check — if `PENotification.launchURL` is referenced without `PushEngageAutoHandleDeeplinkURL` being set and without a handler being registered, flag it.

## IOS-BP-12 — CocoaPods post-install hook (conditional — no longer required for `1.0.0`)

**Rule:** The `APPLICATION_EXTENSION_API_ONLY = 'No'` post-install hook is **no longer required for `1.0.0`**. Extension targets link only the extension-safe `PushEngageExtension` pod, so the original cause is gone. Keep the hook only as a fallback if your `PushEngage` app pod is (incorrectly) linked into an extension target, or if you still see `APPLICATION_EXTENSION_API_ONLY` errors.

**Why:** Earlier baselines shipped a single shared pod, so the app-only API bled into extension targets and tripped Apple's `APPLICATION_EXTENSION_API_ONLY` linker check — the override worked around that. In `1.0.0` the SDK is split into two modules (`PushEngage` for the app, `PushEngageExtension` for NSE/NCE), and the `APPLICATION_EXTENSION_API_ONLY` conditional-compilation mechanism that previously disabled click/view tracking under CocoaPods is removed entirely; tracking is now protected by a `UIApplication` background task in every distribution. With extension targets linking only `PushEngageExtension`, the override is normally unnecessary.

**How to apply:** Audit check IOS-AUDIT-07 is now conditional — do not flag a missing hook as a likely cause on `1.0.0`. The likely cause to look for instead is an extension target still on the `PushEngage` pod/import/prefix (IOS-AUDIT-15). Only suggest the fallback hook if `APPLICATION_EXTENSION_API_ONLY` errors actually surface.

## IOS-BP-13 — Xcode 15+ User Script Sandboxing must be off

**Rule:** Build Settings → User Script Sandboxing → No. Default in Xcode 15+ is Yes, and that default breaks pod install / SDK build phases.

**Why:** The PushEngage podspecs declare no script phases of their own, but CocoaPods' generated build phases (`[CP] Embed Pods Frameworks`, resource copying) need filesystem access broader than the default sandbox allows.

**How to apply:** Mention during install step; audit when build failures present.

## IOS-BP-14 — `setInitialInfo` must run synchronously inside `didFinishLaunchingWithOptions` (IAM background refresh depends on it)

**Rule:** Keep `PushEngage.setInitialInfo(for:with:)` inside `application(_:didFinishLaunchingWithOptions:)` and don't defer it (no `DispatchQueue.async`, no SceneDelegate, no lazy first-screen call). SwiftUI `App`-lifecycle apps need an `@UIApplicationDelegateAdaptor` to have that hook at all — the `init-code.md` templates already do this.

**Why:** From `1.1.0` the in-app-messaging controller is first constructed inside `setInitialInfo`, and it registers the optional `BGTaskScheduler` refresh task at that moment. iOS only accepts launch-handler registration before `didFinishLaunching` returns; anything later means background refresh never registers (and IOS-BP-01's init-order reasoning applies as before).

**How to apply:** Covered by IOS-AUDIT-03 (init order). Additionally flag a `setInitialInfo` call wrapped in an async dispatch, or living outside the AppDelegate, as Worth fixing.

## IOS-BP-15 — IAM background refresh is opt-in and needs both halves

**Rule:** To let in-app campaigns refresh in the background, add **both** `BGTaskSchedulerPermittedIdentifiers` = `["com.pushengage.iam.refresh"]` to Info.plist **and** Background Modes → Background fetch (`UIBackgroundModes` contains `fetch`). Either half alone does nothing useful. Skip both if the customer is happy with refresh-on-app-open.

**Why:** iOS refuses a background task identifier the app hasn't declared, and won't run app-refresh tasks without the fetch mode. With only one half, the SDK logs a single line at launch and falls back to refreshing on app open — not a failure, just wasted config. Scheduling is opportunistic; the ~6 h interval is an earliest-start, never a guarantee.

**How to apply:** Audit check IOS-AUDIT-16 — flag the identifier present without `fetch` (or `fetch` present with the identifier missing while the customer says they want background refresh) as Worth fixing.

## IOS-BP-16 — Register the IAM custom-action handler once, app-wide

**Rule:** Call `PushEngage.setIAMCustomActionHandler` exactly once, in the AppDelegate right after `setInitialInfo`, and dispatch to the app's router from there. Capture `[weak self]` if the closure references a controller.

**Why:** The handler is a single slot — every call replaces the previous closure, there is no getter and no removal, and the closure is retained for the process lifetime. Inside it, route on `parameters["action"]` (the dashboard Action name); `actionId` is an internal dashboard-generated key. A view controller registering its own handler in `viewDidLoad` silently disables the app-level one for every other screen, and campaigns with `custom` buttons can display on any screen.

**How to apply:** Audit check IOS-AUDIT-17 — more than one `setIAMCustomActionHandler` call site, or one outside the AppDelegate, is Worth fixing.

## IOS-BP-17 — Deployment target ≥ iOS 15.0 with SDK `1.1.0`

**Rule:** Before bumping to `1.1.0`, confirm the app (and every extension target linking `PushEngageExtension`) has a deployment target of iOS 15.0 or higher. If the customer must keep iOS 12–14, they stay on `1.0.0` and don't get IAM — say so plainly.

**Why:** `1.1.0` raised the minimum from 12.0 to 15.0 on both pods ("the lowest version Xcode 27 builds for", per the changelog). A lower target fails at `pod install` ("`PushEngage (1.1.0)` … does not support `iOS <target>`") or at SPM resolution (KI-009).

**How to apply:** Audit check IOS-AUDIT-18.

---

Spokes citing iOS rules: just write the rule ID (e.g., "violates IOS-BP-08, missing two of four delegate forwards"). Customer can click into this file for the why.
