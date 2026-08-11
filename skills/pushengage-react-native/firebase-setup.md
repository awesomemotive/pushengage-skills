# Firebase Setup (React Native — Android side)

React Native apps shipping to Android need Firebase Cloud Messaging configured exactly like a native Android app. The full click-path (Firebase Console → register Android app → download `google-services.json` → generate Service Account JSON → capture Sender ID) lives in **`../pushengage-android/firebase-setup.md`**. Read that file first, then return here for the RN-specific deltas below.

> **If the customer is shipping iOS only:** skip this file. Go to `apns-setup.md`.

## What to capture (4 artifacts)

Identical to the Android-native spoke:

1. **`google-services.json`** — downloaded from Firebase Console.
2. **Service Account JSON** — generated from Firebase Settings → Service accounts. **Never ask the customer to paste contents in chat.** They upload directly to PushEngage dashboard.
3. **Sender ID** — from Firebase Settings → Cloud Messaging tab. Numeric.
4. **`applicationId` (package name)** — for cross-checking. In a React Native project, read it from `android/app/build.gradle`'s `defaultConfig.applicationId` line.

## RN-specific deltas

### `google-services.json` placement

```
<RN project root>/
├── package.json
├── android/
│   ├── build.gradle
│   ├── settings.gradle
│   └── app/
│       ├── build.gradle
│       └── google-services.json   ← HERE
└── ios/
```

NOT at the project root. NOT at `android/google-services.json`. Resolves relative to the **app module** (`android/app/`), same as native Android and Flutter. (Same gotcha as KI-A006 / KI-F-equivalent.)

### Where to read the `applicationId` from in an RN project

`android/app/build.gradle`:

```groovy
android {
    defaultConfig {
        applicationId "com.example.myapp"   // ← this value
        // ...
    }
}
```

NOT in `package.json`. NOT in `app.json`. Ask the customer to paste from `android/app/build.gradle`.

### Google Services Gradle plugin version recommendation

The SDK's example app uses `com.google.gms:google-services:4.4.2` in `android/build.gradle` buildscript (bumped from `4.3.15` in SDK 1.0.1). **Recommend `4.4.2`** for new integrations to align with the native Android and Flutter spokes.

### Firebase BOM — customer adds it

The RN plugin does NOT bring Firebase transitively. The customer must add the Firebase BOM + Messaging deps in `android/app/build.gradle`:

```groovy
dependencies {
    implementation platform('com.google.firebase:firebase-bom:33.5.1')
    implementation 'com.google.firebase:firebase-messaging'
}
```

(See `install.md` Step 2c.)

### CRITICAL: Apply the Google Services plugin in `android/app/build.gradle`

**This is the most easily-missed step on RN integrations.** Without `apply plugin: "com.google.gms.google-services"` in the app-level `build.gradle`, `google-services.json` is downloaded but never processed at build time. FCM token issuance silently fails.

Add this line at the top of `android/app/build.gradle` (after `apply plugin: "com.android.application"`):

```groovy
apply plugin: "com.google.gms.google-services"
```

If using the plugins block syntax instead:

```groovy
plugins {
    id "com.android.application"
    id "org.jetbrains.kotlin.android"
    id "com.facebook.react"
    id "com.google.gms.google-services"   // REQUIRED
}
```

Cited as RN-BP-11 / RN-AUDIT-15 / KI-RN010.

### `PENotificationHandlerActivity` registration

The RN plugin's own merged Android manifest contributes the PushEngage notification-handler activity automatically. The customer does NOT need to add `<activity android:name="com.pushengage.pushengage.notificationhandling.PENotificationHandlerActivity" .../>` to their own `android/app/src/main/AndroidManifest.xml`.

## Next

Hand off to `apns-setup.md` if the customer is also shipping iOS, otherwise straight to `dashboard-setup.md`.
