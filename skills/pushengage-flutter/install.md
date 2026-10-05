# Install — Flutter SDK + Native Prerequisites

Install the Flutter plugin and configure native Android + iOS prerequisites. **Authoritative against `version-matrix.md`.** When this file's pins disagree with the public PushEngage Flutter setup doc, this file (and `version-matrix.md`) win.

## Step 1 — `pubspec.yaml`

Add (or update) the dependency:

```yaml
dependencies:
  pushengage_flutter_sdk: ^1.1.0
```

Then:

```bash
flutter pub get
```

> ⚠️ **Use `^1.1.0`** — the current published version (see `version-matrix.md`). Older pins like `^0.0.1` (seen in some earlier setup guides) are missing most of the modern API (`requestNotificationPermission`, `subscribe`, `unsubscribe`, `addSubscriberAttributes`, segments, triggers, etc.), so an app on `^0.0.1` hits "no such method" runtime errors. (FLUTTER-AUDIT-01 / KI-F001.)

If the customer already has `pushengage_flutter_sdk: ^0.0.1` (or any pre-`1.0.0`) pinned, bump it:

```yaml
dependencies:
  pushengage_flutter_sdk: ^1.1.0   # bumped from ^0.0.1 — see KI-F001
```

Then `flutter pub upgrade pushengage_flutter_sdk`.

## Step 2 — Android native configuration

**Skip this step if the customer is shipping iOS-only.**

### Step 2a — `android/settings.gradle` (modern declarative plugins block)

Ensure the Google Services plugin is declared:

```groovy
plugins {
    id "dev.flutter.flutter-plugin-loader" version "1.0.0"
    id "com.android.application" version "8.1.4" apply false
    id "org.jetbrains.kotlin.android" version "1.9.10" apply false
    id "com.google.gms.google-services" version "4.4.2" apply false
}
```

If the customer's `settings.gradle` uses the legacy `buildscript`-style classpath instead of the plugins block, fall back to the legacy form (see `../pushengage-android/install.md` Groovy DSL section). Don't force-migrate to the new style.

### Step 2b — JitPack repository

PushEngage's native Android SDK is on JitPack, not Maven Central. The Flutter plugin's own build script already injects JitPack for classic project-repos builds (the plugin's `android/build.gradle` carries an `allprojects { repositories { maven { url 'https://jitpack.io' } } }` block), so most projects resolve it without edits. Two cases need action:

- **Centralized repos (`dependencyResolutionManagement` in `android/settings.gradle`, especially with `FAIL_ON_PROJECT_REPOS`):** project-level repos are ignored — add JitPack to the `dependencyResolutionManagement { repositories { ... } }` block in `settings.gradle`. This is the case the SDK's own setup doc calls out.
- **Classic mode:** adding it explicitly to `android/build.gradle` is belt-and-braces (harmless, documents the dependency):

```groovy
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }   // REQUIRED for native PushEngage Android SDK
    }
}
```

If the customer uses Kotlin DSL (`android/build.gradle.kts`):

```kotlin
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

> Note: the JitPack dependency is pulled transitively by the Flutter plugin. If `jitpack.io` is missing from every repositories list Gradle consults (e.g., `FAIL_ON_PROJECT_REPOS` without a `settings.gradle` entry), the build fails with "Could not find com.github.awesomemotive:pushengage-android-sdk:1.0.0".

### Step 2c — `android/app/build.gradle` (app-level)

Apply the Google Services plugin in the plugins block:

```groovy
plugins {
    id "com.android.application"
    id "kotlin-android"
    id "com.google.gms.google-services"   // REQUIRED
    id "dev.flutter.flutter-gradle-plugin"
}
```

Add Firebase BOM + Messaging in the dependencies block. Firebase is NOT transitively pulled by the Flutter plugin — the customer adds it:

```groovy
dependencies {
    implementation platform('com.google.firebase:firebase-bom:33.5.1')
    implementation 'com.google.firebase:firebase-messaging'
}
```

> These pins (`google-services:4.4.2`, `firebase-bom:33.5.1`) match the native Android spoke for consistency. The Flutter SDK example app pins `4.4.0` and `33.1.2` — both work, but use these values for new integrations.

### Step 2d — `android/app/src/main/AndroidManifest.xml`

Declare two permissions above the `<application>` element:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

    <application
        android:name="${applicationName}"
        ...>
        ...
    </application>
</manifest>
```

- `INTERNET` — required for the SDK's HTTP calls (typically already present in greenfield projects).
- `POST_NOTIFICATIONS` — declare explicitly for Android 13+ (API 33+). The transitive native SDK already declares it in its library manifest and manifest merger propagates it, so this entry is belt-and-braces (documents intent; protects against `tools:node="remove"` / merger edge cases), not load-bearing. (FLUTTER-AUDIT-05 / KI-F004; mirror of KI-A001.)

The transitive native PushEngage Android SDK's library manifest contributes the PushEngage notification-handler activity (`com.pushengage.pushengage.notificationhandling.PENotificationHandlerActivity`) through manifest merging — the customer does NOT need to add that activity manually. (The Flutter plugin's own manifest is empty.)

### Step 2e — `android/app/google-services.json`

Place the `google-services.json` file (downloaded in `firebase-setup.md`) at `android/app/google-services.json`. NOT at the project root. NOT at `android/google-services.json`. (FLUTTER-AUDIT-06; mirror of KI-A006.)

## Step 3 — iOS native configuration

**Skip this step if the customer is shipping Android-only.**

### Step 3a — `cd ios && pod install` (CRITICAL)

Every `flutter pub get` that touches `pushengage_flutter_sdk` must be followed by:

```bash
cd ios && pod install
```

Flutter's pub resolver does NOT auto-trigger CocoaPods. Skipping this leads to "Pod not found" or "PushEngage pod missing" errors at the next Xcode build. (FLUTTER-BP-09 / KI-F005.)

If `pod install` fails with resolution errors, try:

```bash
pod repo update
pod install
```

If still failing:

```bash
pod deintegrate
pod install
```

### Step 3b — `ios/Podfile`

Set the iOS platform version and add a `post_install` block. Canonical Podfile (the SDK example app also adds an explicit `pod 'PushEngage', '1.1.0'` to `Runner`, which is harmless when pinned to the same version — see Step 3c):

```ruby
platform :ios, '15.0'

ENV['COCOAPODS_DISABLE_STATS'] = 'true'

project 'Runner', {
  'Debug' => :debug,
  'Profile' => :release,
  'Release' => :release,
}

# ... (Flutter's standard flutter_root + require + flutter_ios_podfile_setup boilerplate) ...

target 'Runner' do
  use_frameworks!
  use_modular_headers!

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end
end
```

> As of iOS SDK 1.0.0 the SDK is split into the app pod `PushEngage` and the extension-safe pod `PushEngageExtension`. The extension targets link only `PushEngageExtension`, so the old `APPLICATION_EXTENSION_API_ONLY = 'No'` override is no longer needed — the SDK removed the conditional-compilation mechanism that required it. The `EXCLUDED_ARCHS` and `IPHONEOS_DEPLOYMENT_TARGET` overrides are likewise no longer required (the SDK podspecs carry no arch exclusions and, from `1.1.0`, declare `15.0`).

> Keep `platform :ios, '15.0'` at the top of the Podfile — anything below `'15.0'` no longer resolves against plugin `1.1.0` / `PushEngage 1.1.0` (KI-F013 / FLUTTER-AUDIT-15). If a customer's existing Podfile carries the old overrides, they are harmless but can be removed.

### Step 3c — Adding `pod 'PushEngage'` to the `Runner` target is redundant

The Flutter plugin's own podspec pins `pod 'PushEngage', '1.1.0'` transitively, so you do NOT need to add it to the `Runner` target. The SDK's own example Podfile does include an explicit `pod 'PushEngage', '1.1.0'` in `Runner` and builds fine — an explicit line pinned to the SAME version is harmless. Only remove or avoid it if it pins a DIFFERENT version than the plugin's transitive `1.1.0`, which risks a version mismatch. (FLUTTER-BP-06 / FLUTTER-AUDIT-10.)

NSE and NCE Podfile target blocks need the explicit `pod 'PushEngageExtension', '1.1.0'` line (NOT `pod 'PushEngage'`) — those targets don't go through Flutter's plugin resolution and must link the extension-safe pod. See `nse-setup.md` and `nce-setup.md`.

### Step 3d — `ios/Runner/Info.plist`

Add these keys (most are added automatically by Xcode when enabling Push Notifications + Background Modes capabilities; verify they're present):

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>remote-notification</string>
</array>

<key>PushEngage_App_Group_Key</key>
<string>group.<your-app-bundle>.pushengage</string>

<key>PushEngageAutoHandleDeeplinkURL</key>
<string>NO</string>

<key>PushEngageInAppEnabled</key>
<string>NO</string>
```

- `UIBackgroundModes` — required for silent / data pushes and background image downloads.
- `PushEngage_App_Group_Key` — must match the App Group ID enabled on both the main target and the NSE target (see `nse-setup.md`). Mismatch breaks rich-push analytics (FLUTTER-AUDIT-09 / KI-F007).
- `PushEngageAutoHandleDeeplinkURL` — `"NO"` (default). Set to `"YES"` only if you want http(s) URLs from notifications to auto-open in Safari without going through your handler.
- `PushEngageInAppEnabled` — `"NO"` (default). Toggle when in-app messaging is opted into.

## Step 4 — Pin recap

Authoritative pins (cross-check with `version-matrix.md`):

- **Flutter plugin:** `pushengage_flutter_sdk: ^1.1.0`
- **iOS pod (app, transitive, do not redeclare in `Runner`):** `PushEngage 1.1.0` (requires `platform :ios, '15.0'`)
- **iOS pod (in NSE / NCE targets):** `pod 'PushEngageExtension', '1.1.0'`
- **Android library (transitive):** `com.github.awesomemotive:pushengage-android-sdk:1.0.0`
- **`google-services` plugin:** `4.4.2`
- **Firebase BOM:** `33.5.1`

## Next

Write the init code (`init-code.md`).
