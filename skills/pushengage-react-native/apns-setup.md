# APNs Setup (React Native — iOS side)

React Native apps shipping to iOS need APNs configured exactly like a native iOS app. The full click-path (Apple Developer portal → enable Push Notifications capability on the App ID → Keychain Certificate Signing Request → APNs SSL certificate → `.p12` export with password) lives in **`../pushengage-ios/provider-setup.md`**. Read that file first, then return here for the RN-specific deltas below.

> **If the customer is shipping Android only:** skip this file. Go to `dashboard-setup.md`.

## What to capture (3 artifacts)

Identical to the iOS-native spoke:

1. **`.p12` certificate file** — exported from Keychain after importing the `.cer` from Apple Developer portal.
2. **`.p12` password** — the password chosen during export.
3. **iOS Bundle Identifier** — for the PushEngage dashboard "App Push Id" field.

## RN-specific deltas

### `.p12` not `.p8`

PushEngage's iOS dashboard requires a **`.p12`** certificate, NOT a `.p8` auth key — make sure you export a `.p12`. Follow `../pushengage-ios/provider-setup.md` exactly.

### Where to read the Bundle Identifier from in an RN project

In a React Native project, the iOS Bundle Identifier is set on the Xcode target — NOT in `package.json` or `app.json`:

```
Xcode → open ios/<ProjectName>.xcworkspace
  → select the main app target
  → Signing & Capabilities tab
  → Bundle Identifier field
```

If the customer hasn't opened Xcode yet, they can also read it from `ios/<ProjectName>.xcodeproj/project.pbxproj` (search for `PRODUCT_BUNDLE_IDENTIFIER`). Xcode UI is more reliable.

The Bundle Identifier must match EXACTLY the App ID registered in Apple Developer portal.

### Required iOS capabilities

The Xcode iOS main app target must have these capabilities enabled:

- **Push Notifications** — required for APNs.
- **Background Modes** with `Remote notifications` AND `Background fetch` enabled — required for silent/data pushes and rich-media downloading.
- **App Groups** — required when adding the NSE (see `nse-setup.md`). The App Group ID is created in Apple Developer portal and referenced by both the main app and the NSE.

## Next

Hand off to `dashboard-setup.md`.
