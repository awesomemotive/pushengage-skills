# Notification Service Extension (NSE) Setup

Required for rich notifications (images, mutable content, click tracking). Without it, notifications still arrive — but images aren't attached, and some open analytics don't fire.

## Step 1 — Add the NSE target in Xcode

Auto-create posture: **instruct user via Xcode UI** (Xcode `.pbxproj` files are fragile to edit programmatically).

1. Xcode → **File → New → Target**.
2. Select **Notification Service Extension**.
3. Product Name: `PushEngageNotificationServiceExtension` (or `NotificationService`).
4. Click **Finish**.
5. When Xcode asks **"Activate <name> scheme?"** → **Cancel** (keeps main app as active scheme for debugging).

Ask the customer: "Confirm the new target appears under your project in the Project Navigator."

## Step 2 — Add the SDK dependency to the NSE target

### SPM

1. Xcode → project → NSE target → **General → Frameworks and Libraries**.
2. Click **+** → choose **`PushEngageExtension`** (NOT `PushEngage`) from the existing SPM package.

### CocoaPods

Auto-edit the Podfile — wrap the new NSE target inside the main target block. The app target gets `pod 'PushEngage'`; the NSE target gets `pod 'PushEngageExtension'` (the extension-safe pod). They are different pods now:

```ruby
target 'YourMainTarget' do
  use_frameworks!
  pod 'PushEngage', '1.0.0'

  target 'PushEngageNotificationServiceExtension' do
    pod 'PushEngageExtension', '1.0.0'
  end
end
```

> The `APPLICATION_EXTENSION_API_ONLY = 'No'` post-install hook is **no longer required** for `1.0.0` — the NSE target links only the extension-safe `PushEngageExtension` pod. Keep it only as a fallback if you still hit `APPLICATION_EXTENSION_API_ONLY` errors (see `install.md` and IOS-BP-12).

Then:

```bash
pod install
```

## Step 3 — Replace the generated NotificationService.swift

Auto-edit + announce posture (overwriting a freshly generated template file is low risk).

Replace the contents of `PushEngageNotificationServiceExtension/NotificationService.swift` (or whatever the customer named it) with:

```swift
import UserNotifications
import PushEngageExtension

class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?
    var request: UNNotificationRequest?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler:
                                @escaping (UNNotificationContent) -> Void) {
        self.request = request
        self.contentHandler = contentHandler
        self.bestAttemptContent = request.content.mutableCopy() as? UNMutableNotificationContent

        if let bestContent = bestAttemptContent {
            PushEngageExtension.didReceiveNotificationExtensionRequest(request,
                                                                       bestContentHandler: bestContent)
            contentHandler(bestContent)
        }
    }

    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler,
           let request = request,
           let bestAttemptContent = bestAttemptContent {
            guard let content = PushEngageExtension.serviceExtensionTimeWillExpire(
                    request, content: bestAttemptContent) else {
                contentHandler(bestAttemptContent)
                return
            }
            contentHandler(content)
        }
    }
}
```

(Class name `NotificationService` is the Xcode default — if customer chose a different name, match it.)

## Step 4 — App Groups (required for NSE ↔ main app)

The NSE runs in a separate process and shares state with the main app via App Groups (IOS-BP-10).

### 4.1 — Enable App Groups on BOTH targets

1. Main app target → **Signing & Capabilities → + Capability → App Groups**.
2. Click **+** under App Groups to add a new group.
3. Suggested name: `group.<your.bundle.id>.pushengage` (e.g. `group.com.example.MyApp.pushengage`).
4. NSE target → **Signing & Capabilities → + Capability → App Groups** → check the same group.

### 4.2 — Add the key to BOTH Info.plists

Auto-edit posture: announce + edit.

**Main app `Info.plist`:**

```xml
<key>PushEngage_App_Group_Key</key>
<string>group.com.example.MyApp.pushengage</string>
```

**NSE `Info.plist`:**

```xml
<key>PushEngage_App_Group_Key</key>
<string>group.com.example.MyApp.pushengage</string>
```

The string value must be identical in both. Audit catches mismatches (IOS-BP-10).

## Step 5 — Build and verify

```bash
xcodebuild -workspace YourApp.xcworkspace -scheme YourApp -configuration Debug build | tail -40
```

Expected: succeeds. If the build fails with `no member 'didReceiveNotificationExtensionRequest'` (or similar), the NSE target is still on the old `PushEngage` pod/import/prefix — re-check Steps 2 and 3 (it must use `PushEngageExtension`). If `APPLICATION_EXTENSION_API_ONLY`-related errors still appear (rare with the `1.0.0` split), the fallback post-install hook in IOS-BP-12 / `install.md` applies.

---

Next: if customer wants custom expanded UI → `nce-setup.md`. Otherwise → verification (test push).
