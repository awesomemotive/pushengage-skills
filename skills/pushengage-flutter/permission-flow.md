# Notification Permission Flow (Flutter)

Cross-platform: a single Dart call (`PushEngage.requestNotificationPermission()`) covers both iOS and Android. Per-platform notes below.

## Don't call from `main()` or app-root `initState`

Per FLUTTER-BP-04: defer the request to a meaningful moment — a button tap, the end of onboarding, after the user completes a key first-use step. Don't ask for permission before the user understands why they should grant it.

Both Android (API 33+) and iOS only surface the system permission dialog ONCE in most cases. A denial during cold boot is effectively permanent — recovery requires the user navigating to system settings.

## Canonical snippet

```dart
import 'package:pushengage_flutter_sdk/pushengage_flutter_sdk.dart';

Future<void> requestPushPermission() async {
  final permResult = await PushEngage.requestNotificationPermission();

  if (permResult.data == true) {
    // Permission granted (both platforms).
    final subResult = await PushEngage.subscribe();
    debugPrint('Subscribed: ${subResult.data}');
  } else {
    // Denied, or the request failed (errors surface as success(false),
    // never as a failure — permResult.error is always null).
    debugPrint('Permission not granted');
  }
}
```

Call `requestPushPermission()` from a button's `onPressed`, a "skip / enable" prompt at the end of onboarding, etc.

## Android-specific notes

- **Manifest note:** declare `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` explicitly in `AndroidManifest.xml` (covered in `install.md` Step 2d). The transitive native SDK already declares it and manifest merger propagates it, so the dialog normally appears either way — if it genuinely never appears, check the MERGED manifest for a `tools:node="remove"` before blaming this entry. (FLUTTER-AUDIT-05 / KI-F004; mirror of KI-A001.)
- **Below API 33** (Android 12 and earlier), the permission is granted at install time. The runtime call still serves as the trigger to register the subscriber, so always call it regardless of `targetSdk`.
- **Return semantics:** on Android the returned `bool` reflects the actual grant outcome. `true` = user granted; `false` = user denied.

## iOS-specific notes

- **Return semantics:** as of `1.0.0`, iOS behaves like Android — the Dart future resolves only after the user responds to the system prompt, and the returned `bool` reflects the actual grant outcome. If permission was already decided (granted/denied earlier), the future resolves immediately with the current state.
- **To poll the grant state later** (e.g., on a foreground transition), call `getNotificationPermissionStatus()`:
  ```dart
  final status = await PushEngage.getNotificationPermissionStatus();
  // status.data is "granted" or "denied"
  ```

## Polling the permission status

For UIs that need to drive state based on the current permission grant:

```dart
final status = await PushEngage.getNotificationPermissionStatus();
switch (status.data) {
  case 'granted':
    // Show subscribed UI.
    break;
  case 'denied':
    // Show "Enable notifications in Settings" CTA.
    break;
  default:
    // Treat as denied (defensive).
}
```

`getNotificationPermissionStatus()` returns `"granted"` or `"denied"` on Android. On iOS it can also return `"notYetRequested"` when permission has not been requested yet.

## Re-prompt recovery

Once the user denies the permission, **the only path back is the system settings app on both platforms.** The SDK cannot re-surface the prompt programmatically. Best practice:

1. Detect denial via `getNotificationPermissionStatus()`.
2. Show a one-tap "Open notification settings" CTA.
3. Deep-link to the system settings page via a platform channel (out of scope for this skill to implement — but flag the need to the customer).

The native iOS spoke (`../pushengage-ios/best-practices.md`) and Android spoke (`../pushengage-android/best-practices.md` ANDROID-BP-10) both call out the equivalent recovery pattern. The Flutter implementation typically uses the `app_settings` or `permission_handler` packages to deep-link, but the customer chooses.

## Combined subscription + permission check

If you need to know whether the user can actually receive a push (subscribed AND permission granted), use `getSubscriptionNotificationStatus()`:

```dart
final canReceive = await PushEngage.getSubscriptionNotificationStatus();
// canReceive.data is true iff subscribed AND permission granted.
```

This is the most common gate for showing "you'll receive notifications" UI state.

## Next

Continue to `nse-setup.md` for rich notifications (iOS), or `verification` if the customer is Android-only and doesn't need rich pushes.
