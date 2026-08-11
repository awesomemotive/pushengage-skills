# Notification Content Extension (NCE) Setup — Optional

NCE renders a custom expanded-view UI for PushEngage notifications. **Skipped by default** in the integration flow — only run when the customer explicitly wants custom expanded UI.

## When NOT to do this

If the customer is happy with iOS's default expanded notification view (title + body + image + action buttons), they don't need an NCE. Notifications still deliver normally without it (Service Extension handles rich media). What they lose without an NCE:

- **Only this:** if PushEngage sends a notification with a matching `category`, iOS falls back to its default expanded layout instead of rendering the customer's custom SwiftUI / UIKit view.

Ask first: "Do you need a custom expanded notification UI (your own SwiftUI / UIKit view inside the long-press preview)?" If no → skip this file entirely.

## Step 1 — Add the NCE target

1. Xcode → **File → New → Target → Notification Content Extension**.
2. Click **Next**, name it (e.g. `PushEngageNotificationContentExtension`).
3. When Xcode asks **Activate scheme?** → **Cancel**.
4. Select the top-level project → select the NCE target → set **Deployment Target ≥ iOS 12** (the SDK's minimum). Set it to **≥ iOS 13** if your custom view hosts SwiftUI via `UIHostingController`.

## Step 2 — Set `UNNotificationExtensionCategory` (IOS-BP-09)

This step is easy to miss. **Without it, the NCE never fires.**

Open the NCE's `Info.plist`. Find:

```
NSExtension → NSExtensionAttributes → UNNotificationExtensionCategory
```

The Xcode default is `myNotificationCategory`. **Replace with the category identifier(s) you'll send with custom-UI notifications.** This value must match:
- The `categoryIdentifier` your NCE code checks (Step 4 below).
- The `aps.category` field PushEngage sends from the dashboard for that campaign.

Example:

```xml
<key>UNNotificationExtensionCategory</key>
<string>my_app_rich_content</string>
```

Or, for multiple categories:

```xml
<key>UNNotificationExtensionCategory</key>
<array>
    <string>my_app_rich_content</string>
    <string>my_app_carousel</string>
</array>
```

## Step 3 — Add SDK dependency to NCE target

Same pattern as NSE (Step 2 of `nse-setup.md`) — either SPM (add the **`PushEngageExtension`** product via Xcode UI, NOT `PushEngage`) or CocoaPods (add a nested `target '<NCE target>' do pod 'PushEngageExtension', '1.0.0' end` to the Podfile, then `pod install`).

## Step 4 — Replace the generated NotificationViewController.swift

Auto-edit posture: announce + edit.

```swift
import UIKit
import UserNotifications
import UserNotificationsUI
import PushEngageExtension

class NotificationViewController: UIViewController, UNNotificationContentExtension {

    override func viewDidLoad() {
        super.viewDidLoad()
        self.view.backgroundColor = .white
    }

    func didReceive(_ notification: UNNotification) {
        guard notification.request.content.categoryIdentifier == "my_app_rich_content" else {
            return
        }
        let payload = PushEngageExtension.getCustomUIPayLoad(for: notification.request)
        // `payload` is a `CustomUIModel` — `title`, `body`, `image: UIImage?`,
        // `buttons: [CustomUIButtons]?`. Render it into self.view using whatever
        // UI framework the customer prefers (UIKit views, or SwiftUI via
        // UIHostingController — add `@available(iOSApplicationExtension 13.0, *)`
        // to this class if you host SwiftUI).
        // Example: a UILabel showing payload.title.
    }
}
```

Replace `"my_app_rich_content"` with the customer's actual category identifier from Step 2.

## Step 5 — Build and verify

```bash
xcodebuild -workspace YourApp.xcworkspace -scheme YourApp -configuration Debug build | tail -40
```

Expected: succeeds.

---

Next: verification (test push).
