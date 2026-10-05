# iOS Notification Content Extension (NCE) — React Native

**OPTIONAL.** Default: SKIP. Only run this step when the customer explicitly asks for **custom in-notification UI** (e.g., a SwiftUI view rendered inside the expanded notification view).

> The NCE does NOT enable rich-media notifications — that's the NSE (`nse-setup.md`). The NCE only customizes the expanded view UI when the user long-presses or pulls down a notification banner.

## Quick path

Same pattern as `nse-setup.md`:

1. **File → New → Target → Notification Content Extension** in Xcode.
2. **Product name:** `NotificationContentExtension` (match Podfile target block).
3. **Bundle Identifier:** `<your-main-bundle>.NotificationContentExtension`.
4. **Deployment target:** iOS 15.1 (override Xcode default of 10).
5. **Don't activate scheme** when prompted.
6. Add the NCE target to `ios/Podfile`:
   ```ruby
   target 'NotificationContentExtension' do
     pod 'PushEngageExtension', '1.1.0'
   end
   ```
   **NOT `pod 'PushEngage'`** — as of the 1.0.0 two-pod split, extension targets link the extension-safe `PushEngageExtension` pod. (RN-AUDIT-03.)
7. Run `pod install` from `ios/`.

## Critical gotcha — `UNNotificationExtensionCategory`

The auto-generated NCE `Info.plist` ships with:

```xml
<key>UNNotificationExtensionCategory</key>
<string>myNotificationCategory</string>
```

That's Xcode's placeholder. iOS only routes notifications to the NCE when the notification's `aps.category` matches this value. If left at `myNotificationCategory` and your campaigns send any other category (or none), iOS silently falls back to the default expanded view — the NCE never fires.

**Fix:** replace with the actual category string your campaigns send:

```xml
<key>UNNotificationExtensionCategory</key>
<string>my_app_rich_content</string>
```

Or for multiple:

```xml
<key>UNNotificationExtensionCategory</key>
<array>
    <string>my_app_rich_content</string>
    <string>my_app_carousel</string>
</array>
```

Three things must use the EXACT same string:

1. `UNNotificationExtensionCategory` in NCE `Info.plist`.
2. `notification.request.content.categoryIdentifier` check inside the NCE's `NotificationViewController.didReceive(_:)`.
3. `aps.category` in the campaign payload sent from PushEngage dashboard.

If any one diverges, iOS silently falls back to the default. Mirror of `IOS-AUDIT-05` from iOS-native.

## NCE source template

For the full longer-form template (with `categoryIdentifier` guard, `getCustomUIPayLoad(for:)`, SwiftUI `UIHostingController` wiring), **defer to `../pushengage-ios/nce-setup.md`**. The template is identical to the native iOS case — RN project context doesn't change the NCE internals. As of the 1.0.0 two-pod split the NCE links the extension-safe pod, so the import is `import PushEngageExtension` and the payload call is `PushEngageExtension.getCustomUIPayLoad(for:)`.

## Edit posture

- Xcode UI work — customer-driven.
- NCE `Info.plist` `UNNotificationExtensionCategory` — diff + confirm (critical correctness).
- Podfile NCE target block — auto-edit + announce.

## Next

Verification — real device, dashboard test send, customer confirms.
