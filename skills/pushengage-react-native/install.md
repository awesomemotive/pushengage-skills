# Install — React Native SDK + Native Prerequisites

Install the npm package and configure native Android + iOS prerequisites. **Authoritative against `version-matrix.md`.** When this file's pins disagree with the public PushEngage RN setup doc, this file wins.

## Step 1 — npm install

```bash
npm install @pushengage/pushengage-react-native
# or
yarn add @pushengage/pushengage-react-native
```

> ⚠️ **Use the scoped package name: `@pushengage/pushengage-react-native`.** An unscoped `pushengage-react-native` import (seen in some earlier guides) fails with "Module 'pushengage-react-native' could not be resolved." (RN-AUDIT-01 / KI-RN001.)

### RN version compatibility split

| RN version | What to install |
|---|---|
| `>= 0.78` | Latest: `npm install @pushengage/pushengage-react-native` (resolves to `^1.1.0`). |
| `< 0.78` | **Upgrade RN to 0.78+ first (required).** Every published release from `0.0.2` onward declares a `react-native >= 0.78.0` peer-dependency floor (npm registry). The only exception, `0.0.1` (2025-02, peer dep `react-native: *`), predates the modern API surface and is not a supported path. |

If `package.json` has `@pushengage/pushengage-react-native: ^1.1.0` AND `react-native: <0.78.0`, the iOS build will fail with TurboModuleRegistry errors. (RN-AUDIT-14 / KI-RN009.)

**For old projects:** the only supported path is to upgrade RN to 0.78+. Do not pin `0.0.1` to dodge the peer-dep floor — it predates the modern API (no `identify`/`logout`/`trackEvent`, old `setSmallIconResource` semantics) and is unsupported. (KI-RN009.)

## Step 2 — Android native configuration

**Skip this step if the customer is shipping iOS-only.**

### Step 2a — `android/gradle.properties`

Ensure the New Architecture flag is set:

```properties
newArchEnabled=true
```

(RN 0.78+ defaults this on for new projects; verify it's not been turned off.)

### Step 2b — `android/build.gradle` (project-level)

Add the Google Services plugin classpath + the JitPack repository:

```groovy
buildscript {
    dependencies {
        // ... existing classpaths (com.android.tools.build, kotlin-gradle-plugin) ...
        classpath 'com.google.gms:google-services:4.4.2'
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }   // REQUIRED — native PushEngage Android SDK is on JitPack
    }
}
```

(The example app also uses `4.4.2` since SDK 1.0.1.)

### Step 2c — `android/app/build.gradle` (app-level)

**CRITICAL: apply the Google Services plugin.** Without it, `google-services.json` is downloaded but never read at build time and FCM stays unconfigured.

```groovy
apply plugin: "com.android.application"
apply plugin: "org.jetbrains.kotlin.android"
apply plugin: "com.facebook.react"
apply plugin: "com.google.gms.google-services"   // REQUIRED — RN-BP-11 / KI-RN010
```

(If the project uses the plugins block syntax, use `id("com.google.gms.google-services")` instead.)

Add Firebase BOM + Messaging dependencies — the RN plugin does NOT bring Firebase transitively:

```groovy
dependencies {
    implementation("com.facebook.react:react-android")
    // ... existing RN deps ...
    implementation platform('com.google.firebase:firebase-bom:33.5.1')
    implementation 'com.google.firebase:firebase-messaging'
}
```

### Step 2d — `android/app/src/main/AndroidManifest.xml`

Declare two permissions above the `<application>` element:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

    <application
        android:name=".MainApplication"
        ...>
        ...
    </application>
</manifest>
```

- `INTERNET` — required for the SDK's HTTP calls (typically already present in greenfield RN templates).
- `POST_NOTIFICATIONS` — declare explicitly for Android 13+. The transitive native SDK already declares it in its library manifest and manifest merger propagates it, so this entry is belt-and-braces (documents intent; protects against `tools:node="remove"` / merger edge cases), not load-bearing. **The SDK's example app omits it — add it anyway.** (RN-AUDIT-05 / KI-RN004.)

### Step 2e — `android/app/google-services.json`

Place the file at `android/app/google-services.json`. NOT at the project root. NOT at `android/google-services.json`. (RN-AUDIT-06; mirror of KI-A006.)

## Step 3 — iOS native configuration

**Skip this step if the customer is shipping Android-only.**

### Step 3a — `cd ios && pod install` (CRITICAL)

Every `npm install` / `yarn add` that touches `@pushengage/pushengage-react-native` must be followed by:

```bash
cd ios && pod install
```

npm/yarn don't auto-trigger CocoaPods. Skipping → "Pod not found" / "PushEngage pod missing" build errors. (RN-BP-09 / KI-RN005.)

If `pod install` fails: `pod repo update && pod install`. If still failing: `pod deintegrate && pod install`.

### Step 3b — `ios/Podfile`

Canonical Podfile structure (matches the SDK example app). Critical pieces:

```ruby
ENV['RCT_NEW_ARCH_ENABLED'] = '1'   # required for RN 0.78+

# Standard RN Podfile boilerplate (resolve react_native_pods.rb, platform, prepare_react_native_project!, etc.)

target '<YourAppName>' do
  config = use_native_modules!

  use_react_native!(
    :path => config[:reactNativePath],
    :app_path => "#{Pod::Config.instance.installation_root}/.."
  )

  post_install do |installer|
    react_native_post_install(
      installer,
      config[:reactNativePath],
      :mac_catalyst_enabled => false,
    )

    installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
        # Bump only the PushEngage Swift pods to the iOS-16 floor that links
        # on Xcode 26 (the back-deploy Swift compat lib is gone below 16).
        # The example app scopes this to the PushEngage targets.
        if ['PushEngage', 'PushEngageExtension', 'pushengage-react-native'].include?(target.name)
          config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '16.0'
        end
      end
    end
  end
end

# Per the 1.0.0 migration, extension targets link only the extension-safe
# 'PushEngageExtension' pod (NOT the full app-side 'PushEngage' pod).
target 'NotificationServiceExtension' do
  pod 'PushEngageExtension', '1.1.0'
end

target 'NotificationContentExtension' do
  pod 'PushEngageExtension', '1.1.0'
end
```

**The two-pod split (new in 1.0.0).** The app/main target gets the full `PushEngage` pod transitively through the RN plugin; extension targets (NSE/NCE) link the separate extension-safe `PushEngageExtension` pod. This split replaces the old `APPLICATION_EXTENSION_API_ONLY = 'No'` conditional-compilation mechanism — that override is **obsolete in 1.0.0** and the example app's `post_install` no longer sets it (nor the `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES` override). The only `post_install` setting the example carries is the `IPHONEOS_DEPLOYMENT_TARGET = '16.0'` bump on the PushEngage Swift pods (Xcode-26 link requirement). (RN-AUDIT-04.)

### Step 3c — Do NOT add `pod 'PushEngage'` to the main app target

The RN plugin pins `pod 'PushEngage', '1.1.0'` as a transitive dependency. Redeclaring it in the main `target '<YourAppName>' do` block risks pulling a different version and breaks symbol resolution. (RN-BP-06 / RN-AUDIT-10.)

NSE and NCE target blocks DO need an explicit `pod 'PushEngageExtension', '1.1.0'` line — those targets don't go through RN's plugin resolution, and they link the extension-safe pod, not the app-side `PushEngage` pod.

### Step 3d — `ios/<YourAppName>/Info.plist`

Required keys (most are added automatically by Xcode when enabling capabilities):

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

- `UIBackgroundModes` — required for silent/data pushes and rich-media downloads.
- `PushEngage_App_Group_Key` — must match the App Group ID enabled on both the main target AND the NSE target. Same value in BOTH `Info.plist` files. Mismatch breaks rich-push analytics (RN-AUDIT-09 / KI-RN007).
- `PushEngageAutoHandleDeeplinkURL` — `"NO"` (default).
- `PushEngageInAppEnabled` — `"NO"` (default).

## Step 4 — Pin recap

Authoritative pins (cross-check with `version-matrix.md`):

- **npm package:** `@pushengage/pushengage-react-native: ^1.1.0` (requires RN 0.78+; no supported build for older RN).
- **iOS pod (transitive, do not redeclare in main target):** `PushEngage 1.1.0`.
- **iOS pod (in NSE / NCE targets):** `pod 'PushEngageExtension', '1.1.0'` (the extension-safe pod, NOT `PushEngage`).
- **Android library (transitive):** `com.github.awesomemotive:pushengage-android-sdk:1.0.0`.
- **`google-services` plugin:** `4.4.2`.
- **Firebase BOM:** `33.5.1`.

> No structural install change: JitPack repo still required (Android native SDK still JitPack-hosted), `newArchEnabled=true` / `RCT_NEW_ARCH_ENABLED=1` still required. Firebase BOM / google-services versions are not driven by this SDK bump. The only iOS change is the 1.0.0 two-pod split — extension targets now link `PushEngageExtension` instead of carrying the `APPLICATION_EXTENSION_API_ONLY` override.

## Next

Write the init code (`init-code.md`).
