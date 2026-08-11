# iOS Main Target — Init Code

PushEngage init lives in the main app's `AppDelegate.swift` (UIKit) or in the `UIApplicationDelegateAdaptor` of a SwiftUI app. Init order is fixed (IOS-BP-01): `swizzleInjection` in `init()`, then `setAppID` → `setInitialInfo` → optional handlers → `enableLogging`.

**Edit posture:** diff + confirm. Show the diff to the customer before applying.

**Mode decision (IOS-BP-02):** default = swizzling. Switch to manual only when one of the constraints documented in IOS-BP-02 applies.

---

## Swizzling mode (default) — Swift / UIKit AppDelegate

```swift
import UIKit
import PushEngage

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?

    override init() {
        super.init()
        PushEngage.swizzleInjection(isEnabled: true)
    }

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions:
                     [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        PushEngage.setAppID(id: "<APP_ID_UUID_FROM_DASHBOARD>")
        PushEngage.setInitialInfo(for: application, with: launchOptions)
        #if DEBUG
        PushEngage.enableLogging = true   // XP-BP-07: never on in release
        #endif
        return true
    }
}
```

Replace `<APP_ID_UUID_FROM_DASHBOARD>` with the UUID copied in `dashboard-setup.md` Step 3.

## Swizzling mode (default) — SwiftUI App

```swift
import SwiftUI
import PushEngage

@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

class AppDelegate: NSObject, UIApplicationDelegate {
    override init() {
        super.init()
        PushEngage.swizzleInjection(isEnabled: true)
    }

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions:
                     [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        PushEngage.setAppID(id: "<APP_ID_UUID_FROM_DASHBOARD>")
        PushEngage.setInitialInfo(for: application, with: launchOptions)
        #if DEBUG
        PushEngage.enableLogging = true
        #endif
        return true
    }
}
```

## Swizzling mode (default) — Objective-C

```objectivec
#import "AppDelegate.h"
@import PushEngage;

@implementation AppDelegate

- (instancetype)init {
    self = [super init];
    if (self) {
        [PushEngage swizzleInjectionWithIsEnabled:YES];
    }
    return self;
}

- (BOOL)application:(UIApplication *)application
        didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [PushEngage setAppIDWithId:@"<APP_ID_UUID_FROM_DASHBOARD>"];
    [PushEngage setInitialInfoFor:application with:launchOptions];
#ifdef DEBUG
    [PushEngage setEnableLogging:YES];
#endif
    return YES;
}

@end
```

---

## Manual (non-swizzling) mode — when to use

Only when one of the IOS-BP-02 constraints applies:
- Another SDK already swizzles `UIApplicationDelegate` (e.g., Firebase Messaging with `FirebaseAppDelegateProxyEnabled = YES`).
- Customer policy disallows runtime swizzling.
- Unusual delegate setup where swizzling misbehaves.

State the constraint clearly in chat before switching to this template. Recommend `// MARK: - PushEngage manual integration (reason: <constraint>)` as a code comment so future maintainers understand.

## Manual mode — Swift template

All four delegate methods are required (IOS-BP-08). Missing any silently breaks a piece of the integration.

```swift
import UIKit
import UserNotifications
import PushEngage

@main
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    var window: UIWindow?

    override init() {
        super.init()
        PushEngage.swizzleInjection(isEnabled: false)
    }

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions:
                     [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        PushEngage.setAppID(id: "<APP_ID_UUID_FROM_DASHBOARD>")
        PushEngage.setInitialInfo(for: application, with: launchOptions)
        UNUserNotificationCenter.current().delegate = self
        application.registerForRemoteNotifications()
        return true
    }

    // 1/4 — device token registration
    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        PushEngage.registerDeviceToServer(with: deviceToken)
    }

    // 2/4 — background / silent push
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completionHandler:
                         @escaping (UIBackgroundFetchResult) -> Void) {
        PushEngage.receivedRemoteNotification(application: application,
                                              userInfo: userInfo,
                                              completionHandler: completionHandler)
    }

    // 3/4 — foreground presentation
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                willPresent notification: UNNotification,
                                withCompletionHandler completionHandler:
                                    @escaping (UNNotificationPresentationOptions) -> Void) {
        PushEngage.willPresentNotification(center: center,
                                           notification: notification,
                                           completionHandler: completionHandler)
    }

    // 4/4 — tap / action-button
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                didReceive response: UNNotificationResponse,
                                withCompletionHandler completionHandler: @escaping () -> Void) {
        PushEngage.didReceiveRemoteNotification(with: response)
        completionHandler()
    }
}
```

## Optional handlers

Optional but commonly needed. Add inside `didFinishLaunchingWithOptions` after `setInitialInfo`:

```swift
PushEngage.setNotificationOpenHandler { result in
    // result.notification.additionalData
    // result.notificationAction.actionID
    // navigate based on the actionID / deeplink
}

PushEngage.setNotificationWillShowInForegroundHandler { notification, completion in
    // Decide whether to display the notification while app is active.
    // IOS-BP-05: call completion within 25 seconds.
    completion(notification)  // pass notification = show; pass nil = suppress
}
```

## Permission request

PushEngage auto-subscribes when the system permission is granted. The customer should call:

```swift
PushEngage.requestNotificationPermission { granted, error in
    // optional: react to grant/deny
}
```

…**at a meaningful moment** (XP-BP-03), not on app launch.

## Info.plist additions

| Key | Value | Purpose |
|---|---|---|
| `PushEngageInAppEnabled` | `YES` / `NO` | YES = open http URLs in in-app WKWebView; NO = open in Safari. |
| `PushEngageAutoHandleDeeplinkURL` | `YES` / `NO` | YES = SDK opens deeplinks itself; NO = hand to your `setNotificationOpenHandler`. Default behavior is NO (IOS-BP-11). |

Auto-edit posture: announce + edit (these are inert Info.plist additions, low risk).

---

Next: if rich notifications (images, action buttons) are wanted → `nse-setup.md`. If custom expanded UI → `nce-setup.md`.
