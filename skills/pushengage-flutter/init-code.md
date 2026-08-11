# Init Code (Flutter)

Three pieces of init code: Dart `lib/main.dart`, iOS `ios/Runner/AppDelegate.swift`, and (note) NO Android `Application` subclass needed. Order matters — Dart init triggers native init through the method channel; the Swift side handles delegate-level wiring.

## 1. Dart — `lib/main.dart`

Canonical greenfield template (cross-platform):

```dart
import 'dart:async' show StreamSubscription;
import 'dart:io' show Platform;
import 'package:flutter/foundation.dart' show kDebugMode;
import 'package:flutter/material.dart';
import 'package:pushengage_flutter_sdk/pushengage_flutter_sdk.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();

  // Platform-conditional App ID — one PushEngage site per platform (KI-F006).
  PushEngage.setAppId(
    Platform.isIOS ? '<IOS_APP_ID_UUID>' : '<ANDROID_APP_ID_UUID>',
  );

  // Debug logging — wrap in kDebugMode (FLUTTER-BP-05).
  if (kDebugMode) {
    PushEngage.enableLogging(true);
  }

  runApp(const MyApp());
}
```

**Critical rules:**

1. Prefer calling `setAppId` from `main()` BEFORE `runApp(...)` (FLUTTER-BP-01) so subscriber registration starts as early as possible. Calling it later (e.g. from a widget's `initState()`) is not an error — the SDK example app does this deliberately because it enters the App ID at runtime — but for a greenfield app with a known App ID, `main()` is the cleaner home.
2. `WidgetsFlutterBinding.ensureInitialized()` MUST come before `setAppId` if any plugin-channel call could happen before `runApp`. Cheap insurance; always include it.
3. Replace `<IOS_APP_ID_UUID>` and `<ANDROID_APP_ID_UUID>` with the actual UUIDs captured in `dashboard-setup.md`. Per XP-BP-01, leave a comment noting these should later be moved to a config file or environment-based constants, not hardcoded literals.
4. If the customer is shipping to one platform only, simplify the `setAppId` line:
   - iOS-only: `PushEngage.setAppId('<IOS_APP_ID_UUID>');` (drop the `Platform.isIOS ?` branch).
   - Android-only: `PushEngage.setAppId('<ANDROID_APP_ID_UUID>');`.

### Deep-link listener — register in root widget's `initState`

```dart
class MyApp extends StatefulWidget {
  const MyApp({super.key});
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  late final StreamSubscription _deepLinkSub;

  @override
  void initState() {
    super.initState();
    _deepLinkSub = PushEngage.deepLinkStream.listen((data) {
      if (data == null) return;
      final url = data['deepLink'] as String?;
      if (url == null) return;
      _handleDeepLink(Uri.parse(url));
    });
  }

  void _handleDeepLink(Uri uri) {
    // Customer-specific routing — switch on uri.path, push routes, etc.
  }

  @override
  void dispose() {
    _deepLinkSub.cancel();   // FLUTTER-BP-08
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: Home());
  }
}
```

**Critical rules:**

1. Register the listener in the ROOT widget's `initState`, NOT in a downstream route's `initState`, NOT after auth, NOT inside a `FutureBuilder`. The native side buffers incoming deep links until the first access of `deepLinkStream` signals Dart is ready, then drains the queue into the stream — so the `.listen(...)` must be attached at that first access (as in the snippet: the getter and `.listen` run together). Registering late risks the drain firing into a broadcast stream with no listener yet, and the native buffer is capped (oldest events dropped past 32, both platforms). On iOS, a cold-boot (force-quit) tap is never delivered on this stream at all — it lands in `getInitialNotification()` (below). (FLUTTER-BP-02 / KI-F003.)
2. Cancel the subscription in `dispose()` (FLUTTER-BP-08) — avoids accumulating listeners across hot reloads.
3. Don't pre-empt routing in this file; just capture the URL and dispatch into the customer's existing router.

### Cold-boot recovery — `getInitialNotification()`

On iOS, a notification tap that **cold-launches** the app from a force-quit state fires before Dart can subscribe to `deepLinkStream`, so it is delivered via `getInitialNotification()` instead. Call it once on startup, in addition to listening on `deepLinkStream`. It resolves `null` on Android and on every call after the first (idempotent drain).

```dart
final res = await PushEngage.getInitialNotification();
if (res.isSuccess && res.data != null) {
  // route from the cold-boot notification payload
}
```

This is the Flutter analog of the iOS-native cold-boot fix; it materially mitigates the cold-boot tap loss on iOS (KI-F003). Warm-tap timing and Android still rely on early listener registration (FLUTTER-BP-02), so keep the `deepLinkStream.listen` call too — `getInitialNotification()` complements it, it does not replace it.

## 2. iOS — `ios/Runner/AppDelegate.swift` (swizzling mode, default)

Canonical greenfield template (matches the Flutter SDK example app):

```swift
import Flutter
import UIKit
import PushEngage

@main
@objc class AppDelegate: FlutterAppDelegate {

    override init() {
        super.init()
        PushEngage.swizzleInjection(isEnabled: true)
    }

    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        if #available(iOSApplicationExtension 10.0, *) {
            UNUserNotificationCenter.current().delegate = self
        }
        GeneratedPluginRegistrant.register(with: self)

        PushEngage.setBadgeCount(count: 0)

        PushEngage.setNotificationWillShowInForegroundHandler { notification, completion in
            if notification.contentAvailable == 1 {
                // Silent / data push — let the OS handle it.
                completion(nil)
            } else {
                completion(notification)
            }
        }

        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}
```

**Critical rules:**

1. `swizzleInjection(isEnabled: true)` MUST be inside `override init()`, NOT inside `didFinishLaunchingWithOptions`. The SDK's swizzling needs to register before `UIApplication` queries the delegate for selector availability. Wrong placement = double-init crashes or silent no-op.
2. Do NOT call `PushEngage.setAppId(...)` from Swift. The App ID is set from Dart. Calling it from both sides causes a registration race.
3. Keep `super.application(application, didFinishLaunchingWithOptions: launchOptions)` as the final return — it triggers `FlutterAppDelegate`'s own setup.
4. The `setBadgeCount(0)` and `setNotificationWillShowInForegroundHandler` calls are optional but recommended (matches example app). The latter controls whether banners show while the app is in the foreground — return `notification` to show it, `nil` to suppress.

### Manual (non-swizzling) mode

Use only if a hard constraint forces it (e.g., another iOS SDK already swizzles `UIApplicationDelegate`). Defer to `../pushengage-ios/init-code.md` for the verbatim four-forward template (didRegisterForRemoteNotificationsWithDeviceToken, didReceiveRemoteNotification, willPresent, didReceive). Same four forwards apply — wrap them in `FlutterAppDelegate` subclass overrides and call the iOS-native PushEngage helpers explicitly.

## 3. Android — no `Application` subclass needed

Unlike native Android (where the Android-spoke walks customers through creating a `MyApp : Application()` class to host PushEngage init), the **Flutter plugin handles native Android initialization through its own plugin lifecycle** (the plugin is registered via Flutter's plugin loader).

When Dart calls `PushEngage.setAppId(...)`, the method channel forwards the App ID to the native Android plugin, which initializes the SDK on first call. The customer does NOT need to:

- Create an `Application` subclass.
- Override `Application.onCreate()`.
- Add `android:name=".MyApp"` to `AndroidManifest.xml`.
- Invoke the PushEngage Android SDK's `Builder()` pattern manually.

**If the customer ALREADY has an `Application` subclass for other reasons** (e.g., they use Hilt, set up crash reporting, or seed an in-memory cache), leave it alone. Don't add a PushEngage `Builder()` call to it — that would double-initialize the SDK.

## Edit posture

- `lib/main.dart` — **diff + confirm.** Entry-point code; bad edits break the app.
- `ios/Runner/AppDelegate.swift` — **diff + confirm.** Same reason.
- `ios/Podfile` — auto-edit + announce (covered in `install.md`).
- `android/app/build.gradle`, `android/AndroidManifest.xml` — auto-edit + announce (covered in `install.md`).

## Next

`permission-flow.md` for the runtime permission request flow, then `nse-setup.md` for rich notifications.
