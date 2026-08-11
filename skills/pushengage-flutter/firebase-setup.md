# Firebase Setup (Flutter — Android side)

Flutter apps shipping to Android need Firebase Cloud Messaging configured exactly like a native Android app. The full click-path (Firebase Console → register Android app → download `google-services.json` → generate Service Account JSON → capture Sender ID) lives in **`../pushengage-android/firebase-setup.md`**. Read that file first, then return here for the Flutter-specific deltas below.

> **If the customer is shipping to iOS only (no `android/` folder being used):** skip this file. Go to `apns-setup.md`.

## What to capture (4 artifacts)

Identical to the Android-native spoke:

1. **`google-services.json`** — downloaded when registering the Android app in Firebase Console.
2. **Service Account JSON** — generated from Firebase Settings → Service accounts → Generate new private key. **Never ask the customer to paste this file's contents in chat.** They upload it directly to the PushEngage dashboard.
3. **Sender ID** — copied from Firebase Settings → Cloud Messaging tab. Numeric.
4. **`applicationId` (package name)** — pasted in chat. Reused later to verify the manifest registration and to confirm the Firebase app is registered with the matching package name.

## Flutter-specific deltas

### `google-services.json` placement

```
<flutter project root>/
├── pubspec.yaml
├── android/
│   ├── build.gradle
│   ├── settings.gradle
│   └── app/
│       ├── build.gradle
│       └── google-services.json   ← HERE
└── ios/
```

NOT at `<flutter project root>/google-services.json`. NOT at `<flutter project root>/android/google-services.json`. The `google-services` Gradle plugin resolves the file relative to the **app module** (`android/app/`), same as a native Android project. Misplacing it triggers the same KI-A006 build failure as in native Android — flagged here as FLUTTER-AUDIT-06.

### Where to read the `applicationId` from in a Flutter project

In a Flutter project, the Android `applicationId` lives in `android/app/build.gradle` (or `android/app/build.gradle.kts`):

```groovy
android {
    defaultConfig {
        applicationId = "com.example.myapp"   // ← this value
        // ...
    }
}
```

NOT in `pubspec.yaml`. NOT in `lib/main.dart`. Ask the customer to paste the `applicationId` line. If they pasted the wrong file, redirect to `android/app/build.gradle*`.

The `applicationId` must match EXACTLY what they register in Firebase Console when adding the Android app. A mismatch results in silent FCM token issuance failures (Firebase rejects pushes for the unregistered package).

### Service Account JSON upload destination

The Service Account JSON is uploaded to **the PushEngage dashboard's Android site**, not anywhere in the customer's Flutter project. The file stays on the customer's machine. See `dashboard-setup.md`.

### Notification-handler activity registration

The transitive native PushEngage Android SDK's library manifest already declares the PushEngage notification-handler activity (`PENotificationHandlerActivity`), and the merged manifest at build time contributes it automatically. **Customer does NOT need to add the PushEngage notification-handler activity to their own `android/app/src/main/AndroidManifest.xml`** — doing so is harmless (the manifest merger deduplicates) but unnecessary. The Flutter plugin's own library manifest declares nothing.

## Next

Hand off to `apns-setup.md` if the customer is also shipping iOS, otherwise straight to `dashboard-setup.md`.
