# iOS Notification Service Extension (NSE) — React Native

The NSE is **REQUIRED** for:
- Rich notifications (images, GIFs, action buttons on iOS).
- Accurate delivery analytics on iOS (the OS reports delivery to the NSE, not the main app).
- Mutable-content notifications (any push with `aps.mutable-content: 1`).

> Skip this file if the customer is shipping to Android only.

Same NSE concept as native iOS and Flutter — the Swift extension target lives inside the RN project's `ios/` folder.

## Step-by-step (Xcode UI)

The NSE target must be created through Xcode's UI. Don't edit `.pbxproj` programmatically.

1. **Open the workspace** — `cd ios && open <YourProjectName>.xcworkspace`. NOT `.xcodeproj` — `pod install` generates the workspace, which is what Xcode needs.
2. **File → New → Target...**
3. Pick **Notification Service Extension**. Click Next.
4. **Product name:** `NotificationServiceExtension` (or whatever name you use in the Podfile target block — they MUST match).
5. **Bundle Identifier:** `<your-main-bundle>.NotificationServiceExtension` (Xcode pre-fills correctly).
6. **Language:** Swift.
7. **Deployment target** (in the target's settings after creation): **iOS 15.1** (matches the RN 0.78 baseline). Xcode's default is iOS 10 — override it to 15.1.
8. When Xcode prompts to **"Activate scheme"** — click **Cancel**. (Activating switches the run scheme to the NSE.)
9. Replace the auto-generated `NotificationService.swift` with the PushEngage NSE template (below).
10. Add the NSE target to `ios/Podfile`:
    ```ruby
    target 'NotificationServiceExtension' do
      pod 'PushEngageExtension', '1.0.0'
    end
    ```
    **NOT `pod 'PushEngage'`** — as of the 1.0.0 two-pod split, extension targets link the extension-safe `PushEngageExtension` pod, not the app-side `PushEngage` pod. (RN-AUDIT-03 / KI-RN011.)
11. Run `pod install` from `ios/`.
12. Add **App Groups** capability:
    - In Xcode, NSE target → Signing & Capabilities → **+ Capability** → **App Groups**.
    - Check the SAME App Group ID set on the main app target.
13. Edit the NSE's `Info.plist`:
    - Add key `PushEngage_App_Group_Key` of type String.
    - Value MUST be identical character-for-character to the main app's `Info.plist` value. (RN-AUDIT-09 / KI-RN007.)

## NSE source template

For the 1.0.0 two-pod split: the extension links the `PushEngageExtension` pod, so the import is `import PushEngageExtension` and the SDK calls are prefixed `PushEngageExtension.` (`didReceiveNotificationExtensionRequest` / `serviceExtensionTimeWillExpire`). Replace `NotificationService.swift` with this:

```swift
import UserNotifications
import PushEngageExtension

@available(iOSApplicationExtension 15.1, *)
class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?
    var request: UNNotificationRequest?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.request = request
        self.contentHandler = contentHandler
        self.bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        if let bestContent = bestAttemptContent {
            PushEngageExtension.didReceiveNotificationExtensionRequest(request, bestContentHandler: bestContent)
            contentHandler(bestContent)
        }
    }

    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler,
           let request = request,
           let bestAttemptContent = bestAttemptContent {
            guard let content = PushEngageExtension.serviceExtensionTimeWillExpire(request, content: bestAttemptContent) else {
                contentHandler(bestAttemptContent)
                return
            }
            contentHandler(content)
        }
    }
}
```

Same template as iOS-native and Flutter. The RN project context doesn't change the NSE internals — it's pure iOS extension code.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Build error: "Application extensions cannot use PushEngage" | NSE target links the app-side `pod 'PushEngage'` instead of the extension-safe `pod 'PushEngageExtension'` | Switch the NSE target block to `pod 'PushEngageExtension', '1.0.0'`. The 1.0.0 two-pod split replaces the old `APPLICATION_EXTENSION_API_ONLY = 'No'` override (now obsolete). See `install.md` Step 3b. RN-AUDIT-03. |
| Build error: "Pod 'PushEngageExtension' not found" | Forgot `pod install` after adding the NSE target block | `cd ios && pod install`. |
| Build error: deployment-target mismatch (iOS 10 vs 15.1) | NSE created with Xcode default of iOS 10 | Set NSE deployment target to iOS 15.1 in Xcode (matches RN 0.78 baseline). |
| Rich-push image doesn't display | NSE not present, OR `Podfile` pin is stale / links the wrong pod (`pod 'PushEngage'` instead of `pod 'PushEngageExtension', '1.0.0'` — KI-RN011), OR App Groups misconfigured (KI-RN007) | Walk through all four checks: target exists + builds + App Groups on both targets + Info.plist key identical. |

## Edit posture

- Xcode UI work (target creation, capability enabling) is customer-driven. You instruct; don't auto-edit `.pbxproj`.
- NSE source file content is **auto-edit + announce**. Replace the Xcode-generated `NotificationService.swift` with the template.
- Podfile NSE target block: auto-edit + announce.

## Next

If the customer wants custom in-notification UI → `nce-setup.md`. Otherwise → verification (real device + dashboard test send).
