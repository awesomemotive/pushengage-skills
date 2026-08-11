# iOS Notification Service Extension (NSE) — Flutter

The NSE is **REQUIRED** for:
- Rich notifications (images, GIFs, action buttons in iOS).
- Accurate delivery analytics on iOS (the OS reports delivery to the NSE, not to the main app).
- Mutable-content notifications (any notification with `aps.mutable-content: 1`).

> Skip this file if the customer is shipping to Android only.

Same target type as a native iOS app's NSE, but bundled inside the Flutter project's `ios/` folder.

## Step-by-step (Xcode UI)

The NSE target must be created through Xcode's UI. **Do not edit `.pbxproj` programmatically** — that's fragile and error-prone. Walk the customer through these clicks.

1. **Open the workspace** — `cd ios && open Runner.xcworkspace`. NOT `Runner.xcodeproj` — `pod install` generates the workspace, which is what Xcode needs.
2. **File → New → Target...**
3. Pick **Notification Service Extension**. Click Next.
4. **Product name:** `NotificationServiceExtension` (or match whatever name you'll use in the Podfile target block below — they MUST match).
5. **Bundle Identifier:** `<your-main-bundle>.NotificationServiceExtension` (Xcode pre-fills this; usually correct).
6. **Language:** Swift.
7. **Deployment target (in the target's settings after creation):** **iOS 12**. Xcode's default is iOS 10 — override it. Matches the rest of the project per `install.md`.
8. When Xcode prompts to **"Activate scheme"** — click **Cancel**. (Activating switches the run scheme to the NSE, which makes debugging the main app harder.)
9. Replace the auto-generated `NotificationService.swift` with the PushEngage NSE template (below).
10. Add the NSE target to `ios/Podfile`:
    ```ruby
    target 'NotificationServiceExtension' do
      use_frameworks!
      pod 'PushEngageExtension', '1.0.0'
    end
    ```
11. Run `pod install` from `ios/`.
12. Add **App Groups** capability:
    - In Xcode, select the NSE target → Signing & Capabilities tab → **+ Capability** → **App Groups**.
    - Check the SAME App Group ID you set up for the main `Runner` target (e.g., `group.<your-bundle>.pushengage`).
    - Both targets MUST have the App Group enabled AND checked.
13. Edit the NSE target's `Info.plist`:
    - Add key `PushEngage_App_Group_Key` of type String with the same App Group ID string as the main app's `Info.plist`.
    - These two values MUST be identical character-for-character. If they diverge, the NSE can't read shared state — rich pushes may render but delivery analytics break (FLUTTER-AUDIT-09 / KI-F007).

## NSE source template

Verbatim — replace `NotificationService.swift` in the NSE target:

```swift
import UserNotifications
import PushEngageExtension

@available(iOSApplicationExtension 10.0, *)
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

Same template as the iOS-native spoke's NSE setup. The Flutter project is irrelevant to the NSE's source — it operates entirely in the iOS extension process.

> Under iOS SDK 1.0.0 the extension imports `PushEngageExtension` (NOT `PushEngage`) and calls the helpers with the `PushEngageExtension.` prefix. Method names and signatures are unchanged — only the module/class prefix. The `@available(iOSApplicationExtension 10.0, *)` annotation can stay; it's not load-bearing (the pod's deployment target is 12).

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Build error: "Application extensions cannot use PushEngage" | The NSE block still pins `pod 'PushEngage'` (the non-extension-safe app pod) instead of the extension-safe `PushEngageExtension`. Under iOS SDK 1.0.0 the `APPLICATION_EXTENSION_API_ONLY = 'No'` override is gone (obsolete) — this is now purely a wrong-pod problem. | Switch the NSE block to `pod 'PushEngageExtension', '1.0.0'`, then `cd ios && pod install`. FLUTTER-AUDIT-04. |
| Build error: "Pod 'PushEngageExtension' not found" | Forgot `pod install` after adding the NSE target block, OR the block still references `PushEngage` instead of `PushEngageExtension` | Confirm the block pins `pod 'PushEngageExtension', '1.0.0'`, then `cd ios && pod install`. |
| Build error: "PushEngageExtension was built for iOS X" or arch mismatch | NSE deployment target mismatched with the main app's | Set NSE deployment target to iOS 12 (matches main app per `install.md`). |
| Rich-push image doesn't display | NSE not present, OR NSE present but `Podfile` pin is stale `pod 'PushEngage', '0.0.6'`/`~>0.0.4` instead of `pod 'PushEngageExtension', '1.0.0'` (KI-F011), OR App Groups capability not enabled on both targets, OR `PushEngage_App_Group_Key` mismatch | Walk through all four: target exists + builds + App Groups on both + Info.plist key matches. FLUTTER-AUDIT-03, FLUTTER-AUDIT-09. |

## Edit posture

- Xcode UI work (target creation, capability enabling) is customer-driven. You instruct; don't auto-edit `.pbxproj`.
- NSE source file content is **auto-edit + announce**. Replace the auto-generated `NotificationService.swift` with the template above.
- Podfile NSE target block: auto-edit + announce.

## Next

If the customer wants custom in-notification UI → `nce-setup.md`. Otherwise → verification (real device + dashboard test send).
