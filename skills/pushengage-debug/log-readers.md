# Log Readers — How to Read PushEngage SDK Debug Logs

When the diagnostic tree asks "look in the logs for X," this file says how.

## How to enable verbose logs (iOS)

In AppDelegate:

```swift
#if DEBUG
PushEngage.enableLogging = true
#endif
```

(Customer should already have this from the integration flow; if not, walk them through adding it.)

## Where logs appear

- Xcode console (when running attached) — most common.
- Console.app on Mac (when running from device, not attached).
- Note: the SDK logs via Swift `print` (attached-console only, format `<ClassName>: <message>`, e.g. `PEManager: …`). A single error path uses `os_log` with the **default** subsystem and a `[PushEngage]` prefix — there is no `com.pushengage` subsystem, so `log stream` predicates on subsystem match nothing; in Console.app search for `[PushEngage]` or SDK class names (`PEManager`, `SubscriberServiceManager`).

## Key log markers to look for

### "APNs token received"

The SDK has no dedicated APNs-token log line. Token receipt is observable indirectly: right after Apple hands over a token, the SDK fires the `subscriber/add` network call, and with logging on the request/response is printed (`Request URL : …subscriber/add`, then `Response status code : …`) — the raw token also appears inside the printed `Request Body JSON` as the payload's endpoint field. If APNs registration **fails**, the error is logged as `ApplicationService: <error message>`. If neither appears, the device didn't get a token from Apple — root cause is APNs reachability or capability misconfig.

### "Subscriber registered"

Look for the network log pair for the add-subscriber call: `Request URL : …subscriber/add` followed by `Response status code : 200` (the endpoint is `subscriber/add`, not `/subscribe`). A non-200 status, or an error line from `SubscriberServiceManager`, means the SDK saw the token but the server call failed. Common causes: wrong App ID (UUID), network failure, server-side rate limit.

### "Notification received"

When a push arrives, the SDK logs receipt. If campaign appears "Delivered to APNs" on the dashboard but no "Notification received" log appears → device-side filter (Branch 1.3.1).

## What NOT to share

If asking the customer to paste logs in chat, instruct them to **redact**:

- The device token (hex string ~ 64 chars).
- The Subscriber ID / hash (if it appears in logs).
- The App ID UUID (less sensitive, but habit).

## When logs are insufficient

PushEngage SDK logs are minimal compared to, e.g., Firebase Messaging. If the customer needs deeper insight:

- For APNs delivery → check Apple Developer → Push Notifications Console.
- For server-side delivery → ask the customer to enable verbose dashboard logs (Site Settings → Logs).

---

## How to enable verbose logs (Android)

In `Application.onCreate()`, gated by `BuildConfig.DEBUG`. This snippet only flips the logging flag — for SDK init (the Builder pattern) see `pushengage-android/init-code.md`.

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            PushEngage.enableLogging(true)
        }
        // ... then the canonical Builder() init from init-code.md ...
    }
}
```

Java equivalent:

```java
if (BuildConfig.DEBUG) {
    PushEngage.enableLogging(true);
}
```

For debug sessions, call `enableLogging` **before** the `PushEngage.Builder()` init call so the init path itself is logged. The canonical init order in `init-code.md` (init → enableLogging) is correct for production; flip the order temporarily during debugging.

## Where Android logs appear

- `adb logcat` from a terminal connected to the device or emulator.
- Filter to PushEngage's tag: `adb logcat -s PushEngage` (single tag).
- Or filter to PushEngage + Firebase Messaging together: `adb logcat -s PushEngage:V FirebaseMessaging:V`.
- Android Studio → Logcat tab also works; set the filter to `tag:PushEngage`.

If `adb` is not on PATH, it lives at `~/Library/Android/sdk/platform-tools/adb` (macOS) or `%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe` (Windows).

## Key Android log markers

### "FCM token received"

On first registration the SDK logs the raw token by itself under tag `PushEngage` (a long bare base64-ish line — `Log.d(TAG, token)`). On a token refresh it logs `onNewToken: upgrading subscriber with new FCM token: <token>`. Failures log `Firebase token retrieval failed` or `FirebaseMessaging unavailable — google-services.json missing or invalid. Subscribe aborted.` If no token line **ever appears**:

- Device has no Google Play Services (bare AOSP emulator) — see diagnostic-tree.md Branch 1.5.1.
- `google-services.json` missing / at wrong path — see KI-A006.
- `com.google.gms.google-services` plugin not applied — see KI-A007.
- Network blocked (corp Wi-Fi, VPN, firewall blocking FCM ports).

### "Subscriber registered"

The SDK logs subscribe **failures** explicitly (`Subscribe API failed with response code: …`, `Add subscriber failed`) and logs `Notification permission granted, calling add subscriber.` on the success path. With logging enabled, the HTTP traffic to `subscriber/add` is also logged by OkHttp under its own tag — include it in the filter: `adb logcat -s PushEngage:V okhttp.OkHttpClient:V`. If the FCM token line appears but the `subscriber/add` call fails or never fires:

- Wrong App ID UUID passed to `setAppId`.
- Sender ID mismatch between Firebase Console and PushEngage dashboard (Branch 1.5.3).
- Network failure between device and PushEngage backend.

### "Notification received"

When a push arrives, `PEFirebaseMessagingService.onMessageReceived` logs `From: <sender>` and, for data payloads, `RemoteMessage data payload: …` under tag `PushEngage`. If the campaign shows "Delivered" on the dashboard but these lines never appear:

- OEM background restriction killed the FCM service (KI-A003 / Branch 7).
- App force-stopped by user (Android won't deliver FCM to force-stopped apps).
- Battery optimization / Doze mode.

## What NOT to share (Android)

When asking the customer to paste `adb logcat` output, instruct them to redact:

- **FCM device token** (long base64-ish string, ~150+ chars). Treat as a credential.
- **App ID UUID** (less sensitive, but redact by habit).
- **Service Account JSON** contents if any line dumps them — these are private keys; never paste in chat.
- Any `Authorization:` header values, OAuth tokens, or other secrets if visible in HTTP debug lines.

---

## How to enable verbose logs (Flutter)

In `lib/main.dart`, gated by `kDebugMode`:

```dart
import 'package:flutter/foundation.dart' show kDebugMode;
import 'package:pushengage_flutter_sdk/pushengage_flutter_sdk.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  if (kDebugMode) {
    PushEngage.enableLogging(true);
  }
  PushEngage.setAppId(...);
  runApp(...);
}
```

This toggles BOTH the Dart-side `DebugLogger` (which `print`s to `flutter logs` / `flutter run` console) and the native logging on each platform via the method channel.

## Where Flutter logs appear

- **Dart-side logs:** Flutter's normal log streams.
  - `flutter logs` from a terminal.
  - `flutter run`'s attached console (when running with the IDE or `flutter run --debug`).
  - IDE's Run/Debug console (VS Code, Android Studio, IntelliJ).
- **iOS native logs:** Xcode console when running through Xcode, or `Console.app` for unattached device runs. Native iOS lines print as `<ClassName>: <message>` (e.g. `PEManager: …`); only the rare error-level `os_log` line carries a `[PushEngage]` prefix — same formats as the iOS section above.
- **Android native logs:** `adb logcat -s PushEngage` (or Android Studio Logcat tab). Same Android-native filtering rules apply.

## Key Flutter log markers

### Method-channel call markers — `PushEngage#<method>`

With logging on, every native invocation logs as a method-channel call. Useful for tracing which Dart calls reached native code.

Grep patterns:
```bash
# In flutter run console
grep 'PushEngage#'

# Or in attached IDE console — search for "PushEngage#"
```

Expected sequences:
- App launch: `PushEngage#setAppId` (Dart called native).
- Permission request: `PushEngage#requestNotificationPermission`.
- Deep link arrival: a method-channel call IN the OTHER direction — `onDeepLink` from native to Dart (visible in Dart-side log as `Data from deeplink: <map>`).

If `PushEngage#setAppId` never appears in logs → `setAppId` was never called from Dart. Common cause: `setAppId` called from a widget's `initState` that never ran (e.g., the widget was never built because of an early return), or the app crashed before the call.

### "App Id set successfully"

Dart-side `DebugLogger.log` confirmation that `setAppId`'s method-channel call returned without throwing. Look in the Dart console.

If you see `Failed to set AppId: <message>` instead — the native side rejected the App ID. Typically malformed UUID, empty string, or a corrupted/uninitialized native plugin (rare; usually a `pod install` skip on iOS — see KI-F005).

### "Data from deeplink: <map>"

Dart-side log emitted when the native side dispatches `onDeepLink` to Dart and the Dart `deepLinkStream` listener picks it up. If this log appears but the customer's app didn't navigate → the issue is in their deep-link handler logic, not in the SDK delivery.

If the customer expects a deep link and this log does NOT appear after a notification tap → either:
- `deepLinkStream.listen(...)` wasn't called yet (FLUTTER-BP-02 / KI-F003) — see diagnostic-tree Branch 3.4.
- The notification payload didn't include a deep link field (`pe.dl` / `pe.u`). Compare against a working notification's `userInfo`.

## What NOT to share (Flutter)

Same redactions as iOS and Android, plus:

- Dart `print` / `debugPrint` output of subscriber maps from `getSubscriberDetails(...)` — they include the subscriber hash and any attributes. Treat the whole map as PII.

---

## How to enable verbose logs (React Native)

In `index.js` (or `index.tsx`), gated by `__DEV__`:

```js
import PushEngage from '@pushengage/pushengage-react-native';

if (__DEV__) {
  PushEngage.enableLogging(true);
}

PushEngage.setAppId('...');
AppRegistry.registerComponent(appName, () => App);
```

This toggles the **native** SDK logging on each platform via the Turbo Module bridge. The RN module itself has no JS-side logger — all SDK diagnostic output appears in the native logs (Xcode console / logcat), not in Metro.

## Where RN logs appear

- **JS-side `console.log` / `debugPrint`:** Metro's console (`npx react-native log-ios` / `log-android` from a terminal, or the Metro process itself).
- **iOS native logs:** Xcode console (when running through Xcode), `Console.app` for unattached device runs. Native iOS lines print as `<ClassName>: <message>` (e.g. `PEManager: …`) — same format as the iOS and Flutter sections above.
- **Android native logs:** `adb logcat -s PushEngage` (or Android Studio Logcat tab; filter to `tag:PushEngage`). Same as Android-native.

## Key React Native log markers

### Turbo Module bridge markers

The RN module does not log its own bridge invocations — there are no per-call markers. The Turbo Module name is `PushengageReactNative`; it shows up in TurboModuleRegistry / codegen error messages (KI-RN009), which is what to grep for when the module fails to load:

```bash
npx react-native log-ios | grep "PushengageReactNative"
# or
npx react-native log-android | grep "PushengageReactNative"
```

To verify which JS calls reached native code, enable native logging (`enableLogging(true)`) and watch the native SDK logs (iOS/Android sections above):
- `setAppId` → triggers the SDK's site-config / `subscriber/add` network activity in the native logs.
- (After permission grant) → subscribe is auto-called by the native SDK.

If no PushEngage native log activity ever appears after launch → `setAppId` was never called from JS. Most common cause: customer put `setAppId` in `App.tsx`'s `useEffect` instead of `index.js` (RN-AUDIT-02 / RN-BP-01).

### "Subscriber registered"

Use the native markers: on iOS the `subscriber/add` request/response network logs (`Request URL : …subscriber/add`, `Response status code : 200`); on Android the `Notification permission granted, calling add subscriber.` line plus the absence of `Subscribe API failed with response code: …`. RN doesn't add its own prefix to those — see the iOS and Android sections above.

### Deep link arrival

When `onValueChanged` fires:
- Native side dispatches to the bridge.
- JS-side `console.log` from inside the customer's callback prints (if they have logging there).

If the JS callback never fires but a notification with a deep link DID arrive on the device → the issue is listener registration timing (KI-RN003 / Branch 3.5).

## What NOT to share (React Native)

Same redactions as iOS and Android, plus:

- JS-side `console.log` output of `getSubscriberDetails(...)` results — include subscriber hash + attributes. Treat the whole map as PII.
- Bridge debug output showing the App ID UUID being passed across — not a secret, but redact by habit.
