# APNs Setup (Flutter — iOS side)

Flutter apps shipping to iOS need APNs configured exactly like a native iOS app. The full click-path (Apple Developer portal → enable Push Notifications capability on the App ID → Keychain Certificate Signing Request → APNs SSL certificate → `.p12` export with password) lives in **`../pushengage-ios/provider-setup.md`**. Read that file first, then return here for the Flutter-specific deltas below.

> **If the customer is shipping to Android only (no `ios/` folder being used, or `ios/` is empty):** skip this file. Go to `dashboard-setup.md`.

## What to capture (3 artifacts)

Identical to the iOS-native spoke:

1. **`.p12` certificate file** — exported from Keychain after importing the downloaded `.cer` from Apple Developer portal.
2. **`.p12` password** — the password the customer chose when exporting.
3. **iOS Bundle Identifier** — captured for the PushEngage dashboard (it goes in the "App Push Id" field).

## Flutter-specific deltas

### `.p12` not `.p8`

PushEngage's iOS dashboard requires a **`.p12`** certificate, NOT a `.p8` auth key — make sure you export a `.p12`. Follow `../pushengage-ios/provider-setup.md` exactly.

### Where to read the Bundle Identifier from in a Flutter project

In a Flutter project, the iOS Bundle Identifier is set on the Xcode `Runner` target — NOT in `pubspec.yaml`:

```
Xcode → open ios/Runner.xcworkspace
  → select Runner target
  → Signing & Capabilities tab
  → Bundle Identifier field
```

If the customer hasn't opened Xcode yet, they can also read it from `ios/Runner.xcodeproj/project.pbxproj` (search for `PRODUCT_BUNDLE_IDENTIFIER`), but the Xcode UI is the more reliable path.

The Bundle Identifier must match EXACTLY what they registered in Apple Developer portal when creating the App ID. A mismatch causes APNs to silently fail (Apple rejects tokens for the unregistered bundle).

### Apple Developer portal capabilities

The Apple Developer portal App ID must have these capabilities enabled:
- **Push Notifications** — required.
- **App Groups** — required if the customer plans to add the NSE (which is required for rich pushes; see `nse-setup.md`). The App Group is created in the portal and referenced by both the main app and the NSE; the configuration in Xcode is covered in `install.md` and `nse-setup.md`.

### Background modes in Xcode

The iOS target also needs the **Background Modes** capability with `Remote notifications` AND `Background fetch` enabled. This is set in Xcode (Signing & Capabilities → + Capability → Background Modes). The corresponding `Info.plist` entry is added automatically — see `install.md` for the resulting `UIBackgroundModes` key.

## Next

Hand off to `dashboard-setup.md`.
