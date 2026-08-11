# iOS Notification Content Extension (NCE) — Flutter

**OPTIONAL.** Default: SKIP. Only run this step when the customer explicitly asks for **custom in-notification UI** (e.g., a SwiftUI view rendered inside the expanded notification view, instead of the system's default).

> The NCE does NOT enable rich-media notifications — that's the NSE (`nse-setup.md`). The NCE only customizes the expanded view UI when the user long-presses or pulls down a notification banner.

## Quick path

Same pattern as `nse-setup.md`:

1. **File → New → Target → Notification Content Extension** in Xcode.
2. **Product name:** `NotificationContentExtension` (or match the Podfile target block name).
3. **Bundle Identifier:** `<your-main-bundle>.NotificationContentExtension`.
4. **Deployment target:** iOS 12 (override Xcode default of 10).
5. **Don't activate scheme.**
6. Add the NCE target to `ios/Podfile`:
   ```ruby
   target 'NotificationContentExtension' do
     use_frameworks!
     pod 'PushEngageExtension', '1.0.0'
   end
   ```
7. Run `pod install` from `ios/`.

## Critical gotcha — `UNNotificationExtensionCategory`

The auto-generated NCE `Info.plist` ships with this entry:

```xml
<key>UNNotificationExtensionCategory</key>
<string>myNotificationCategory</string>
```

That's Xcode's placeholder. iOS only routes a notification to the NCE when the notification's `aps.category` field matches this value. If you leave `myNotificationCategory` here and your campaigns send any other category (or no category at all), iOS silently falls back to the default expanded view — the NCE never fires.

**Fix:** replace `myNotificationCategory` with the actual category string your campaigns send via `aps.category`:

```xml
<key>UNNotificationExtensionCategory</key>
<string>my_app_rich_content</string>
```

Or for multiple categories:

```xml
<key>UNNotificationExtensionCategory</key>
<array>
    <string>my_app_rich_content</string>
    <string>my_app_carousel</string>
</array>
```

Three things must use the EXACT same string:

1. `UNNotificationExtensionCategory` in the NCE `Info.plist` (above).
2. `notification.request.content.categoryIdentifier` check inside the NCE's `NotificationViewController.didReceive(_:)` (Swift code).
3. `aps.category` in the campaign payload sent from the PushEngage dashboard.

If any one diverges, iOS does exactly what's described above. Mirror of `IOS-AUDIT-05` from the iOS-native spoke.

## NCE source template

For the full longer-form template (with `categoryIdentifier` guard, `getCustomUIPayLoad(for:)`, and SwiftUI `UIHostingController` wiring), **defer to `../pushengage-ios/nce-setup.md`**. The template is identical to the native iOS case — the Flutter project context doesn't change the NCE's internals.

> Under iOS SDK 1.0.0 the NCE imports `PushEngageExtension` and calls `PushEngageExtension.getCustomUIPayLoad(for:)` (NOT `PushEngage.getCustomUIPayLoad`) — the extension-safe pod, same as the NSE. The long-form `../pushengage-ios/nce-setup.md` template must be kept in sync with this prefix in the same SDK cycle, or this note will diverge from the file it points to.

## Edit posture

- Xcode UI work — customer-driven.
- NCE Info.plist `UNNotificationExtensionCategory` — diff + confirm (it's a critical-correctness setting).
- Podfile NCE target block — auto-edit + announce.

## Next

Verification step — real device, dashboard test send, customer confirms.
