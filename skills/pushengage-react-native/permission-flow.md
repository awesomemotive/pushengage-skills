# Notification Permission Flow (React Native)

Cross-platform: a single JS call (`PushEngage.requestNotificationPermission()`) covers both iOS and Android. The Promise resolves with the actual grant outcome on both platforms, and on grant the SDK auto-calls `subscribe()`.

## Don't call from `index.js` or root `useEffect`

Per RN-BP-04: defer the request to a meaningful moment — a button tap, the end of onboarding, after a key first-use step. Don't ask for permission before the user understands why they should grant it.

Both Android (API 33+) and iOS only surface the system permission dialog ONCE in most cases. Denial during cold boot is effectively permanent — recovery requires the system settings app.

## Canonical snippet

```tsx
import PushEngage from '@pushengage/pushengage-react-native';

async function requestPushPermission() {
  try {
    const granted = await PushEngage.requestNotificationPermission();

    if (granted) {
      // On grant, the SDK auto-calls subscribe() — no need to call it again.
      console.log('Permission granted. Subscriber registration started.');
    } else {
      console.log('Permission denied.');
      // Show "Enable in Settings" CTA.
    }
  } catch (err) {
    // The native side rejects with a plain Error carrying a string `.code`
    // and `.message`. There is no PushEngagePermissionError instance at
    // runtime — that class is not exported from the package.
    console.error(
      'PushEngage permission error',
      err.code,
      err.message,
    );
  }
}
```

Call from a button's `onPress` handler or a "skip / enable" prompt at end of onboarding.

## Critical rules

1. **The SDK auto-calls `subscribe()` on grant.** Don't call `PushEngage.subscribe()` yourself after `requestNotificationPermission()` returns `true` — double-call is a race.
2. **Wrap the call in `try/catch`** to surface platform-specific error codes. On failure the Promise rejects with a plain `Error` carrying a string `.code` and `.message` — there is no `PushEngagePermissionError` instance at runtime (that class is not exported from `@pushengage/pushengage-react-native`).

## Android-specific notes

- **Manifest note:** declare `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>` explicitly in `AndroidManifest.xml` (covered in `install.md` Step 2d). The transitive native SDK already declares it and manifest merger propagates it, so the dialog normally appears either way — if it genuinely never appears, check the MERGED manifest for a `tools:node="remove"` before blaming this entry. (RN-AUDIT-05 / KI-RN004.)
- **Below API 33** (Android 12 and earlier), the permission is granted at install time; the runtime call still serves as the subscribe trigger.

## iOS-specific notes

- **Return semantics:** the Promise resolves with the actual grant outcome (`true` if granted, `false` if denied). This is more reliable than Flutter's iOS path (which returns `true` regardless).
- **Background errors:** if the system surfaces an error during the prompt (rare), the Promise rejects with a plain `Error` carrying a string `.code` and `.message` (no `PushEngagePermissionError` instance exists at runtime — that class is not exported). Always wrap in try/catch.

## Checking permission status separately

```tsx
const status = PushEngage.getNotificationPermissionStatus();
// status is one of: 'granted' | 'denied' | 'notYetRequested'  (iOS only has the third)
```

This is **synchronous** (not a Promise) on both platforms.

**Status values:**

| Value | Platforms | Meaning |
|---|---|---|
| `'granted'` | Both | User authorized notifications. |
| `'denied'` | Both | User denied (or never prompted on Android API <33 and turned off in settings). |
| `'notYetRequested'` | **iOS only** | The system prompt has never been shown for this app yet. (Android: pre-prompt state resolves to either granted-at-install or you-haven't-asked-yet collapsing into `'denied'`.) |

Handle all three values:

```tsx
const status = PushEngage.getNotificationPermissionStatus();
switch (status) {
  case 'granted':
    // Show subscribed UI.
    break;
  case 'denied':
    // Show "Enable in Settings" deep-link CTA.
    break;
  case 'notYetRequested':
    // iOS-only — show "Enable notifications" button that calls requestNotificationPermission().
    break;
  default:
    // Defensive — treat as denied.
}
```

## Combined subscription + permission check

If you need to know whether the user can actually receive a push (subscribed AND permission granted):

```tsx
const canReceive = await PushEngage.getSubscriptionNotificationStatus();
// canReceive is true iff subscribed AND permission granted.
```

This is the most common gate for showing "you'll receive notifications" UI.

## Re-prompt recovery

Once the user denies, **the only path back is the system settings app** on both platforms. The SDK cannot programmatically re-surface the prompt.

Best practice:

1. Detect denial via `getNotificationPermissionStatus()`.
2. Show a one-tap "Open notification settings" CTA.
3. Deep-link to system settings using `Linking.openSettings()` from `react-native`:

```tsx
import { Linking } from 'react-native';
// ...
<Button title="Enable notifications in Settings" onPress={() => Linking.openSettings()} />
```

`Linking.openSettings()` opens the app's notification settings page on both iOS and Android.

## Next

Continue to `nse-setup.md` for rich notifications (iOS), or `verification` if the customer is Android-only and doesn't need rich pushes.
