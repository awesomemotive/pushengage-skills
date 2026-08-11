# Android — Notification Permission Flow (Android 13+ / API 33+)

`PushEngage.requestNotificationPermission(...)` is the entry point. It triggers the system permission dialog (API 33+) and auto-subscribes the device on grant.

**Audit links:** ANDROID-BP-01, ANDROID-BP-02, ANDROID-BP-03, ANDROID-BP-10.

---

## Hard requirement — `ComponentActivity` (ANDROID-BP-01)

The SDK signature is:

```java
public static void requestNotificationPermission(
    ComponentActivity activity,
    PushEngagePermissionCallback callback
)
```

> ⚠️ **Plain `android.app.Activity` won't compile.** The call site `PushEngage.requestNotificationPermission(this, ...)` where `this` is a plain `Activity` produces a compile error like `requestNotificationPermission(androidx.activity.ComponentActivity, ...) cannot be applied to (com.example.MyActivity)`. KI-A002 / ANDROID-AUDIT-02 / ANDROID-BP-01.

Customer's `MainActivity` (or whichever activity requests permission) must extend one of:

- `androidx.activity.ComponentActivity` (the base).
- `androidx.fragment.app.FragmentActivity` (extends `ComponentActivity`).
- `androidx.appcompat.app.AppCompatActivity` (extends `FragmentActivity` — most common in Java/Kotlin apps).

Plain `android.app.Activity` is **not** in this hierarchy.

### How to detect the base class

```bash
grep -rn "class .*Activity" app/src/main/java app/src/main/kotlin | grep -E "extends Activity\b|: Activity\("
```

If any matches, that activity won't compile as-is when wired to PushEngage.

### Fix

Change the base class:

```kotlin
// Before
class MainActivity : Activity() { ... }

// After
class MainActivity : AppCompatActivity() { ... }
```

```java
// Before
public class MainActivity extends Activity { ... }

// After
public class MainActivity extends AppCompatActivity { ... }
```

`AppCompatActivity` requires `androidx.appcompat:appcompat` on the classpath, which is in nearly every Android project. If not present, add `implementation("androidx.appcompat:appcompat:1.7.0")` (or current).

---

## Manifest prerequisite — `POST_NOTIFICATIONS`

The SDK's library manifest declares `POST_NOTIFICATIONS` and manifest merger propagates it into the app, so the dialog normally appears even without an app-level entry. Still declare it explicitly in `AndroidManifest.xml` for clarity (KI-A001 / ANDROID-AUDIT-01 / ANDROID-BP-02). If the dialog genuinely never appears, check the MERGED manifest before blaming this entry — a `tools:node="remove"` or blocked merger is the rare real cause.

`install.md` adds this entry. If you're authoring permission flow in isolation, confirm the manifest has it.

---

## Kotlin call site

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ... your normal onCreate setup ...
    }

    // ANDROID-BP-03: call this from a user-driven moment (e.g. an
    // "Enable notifications" button), not from onCreate.
    private fun requestPushPermission() {
        PushEngage.requestNotificationPermission(
            this,
            object : PushEngagePermissionCallback {
                override fun onPermissionResult(granted: Boolean, error: Error?) {
                    if (granted) {
                        // SDK auto-subscribes; nothing else to do.
                    } else {
                        // No automatic second prompt path on Android.
                        // Show a button that deep-links to Settings.
                        showEnableInSettingsCTA()
                    }
                }
            }
        )
    }
}
```

## Java call site

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // ... your normal onCreate setup ...
    }

    // ANDROID-BP-03: call from a user-driven moment.
    private void requestPushPermission() {
        PushEngage.requestNotificationPermission(this, new PushEngagePermissionCallback() {
            @Override
            public void onPermissionResult(boolean granted, Error error) {
                if (granted) {
                    // SDK auto-subscribes.
                } else {
                    showEnableInSettingsCTA();
                }
            }
        });
    }
}
```

---

## Meaningful moment (ANDROID-BP-03)

Do NOT call `requestNotificationPermission` in:

- `Application.onCreate()` — won't even compile (no Activity context).
- `MainActivity.onCreate()` — compiles, but the system dialog appears before the user has seen any value from your app. Most users deny. Once denied, Android blocks subsequent prompts (see below).

**Do** call it in response to a user-driven moment:

- Tap on an "Enable notifications" button after onboarding.
- After the user completes a relevant action (e.g. "Notify me when this is in stock").
- After explaining what they'll be notified about.

Mirrors XP-BP-03 from the cross-platform best-practices.

---

## Denial recovery — `Settings.ACTION_APP_NOTIFICATION_SETTINGS` (ANDROID-BP-10)

Once the user denies the system permission, Android suppresses subsequent `requestNotificationPermission` calls — the dialog will not reappear regardless of how many times the SDK is called. The only path back is sending the user to **Settings → Apps → \<App\> → Notifications**.

Provide a one-tap deep link.

### Kotlin

```kotlin
fun openAppNotificationSettings(activity: Activity) {
    val intent = Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).apply {
        putExtra(Settings.EXTRA_APP_PACKAGE, activity.packageName)
    }
    activity.startActivity(intent)
}
```

### Java

```java
public static void openAppNotificationSettings(Activity activity) {
    Intent intent = new Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS);
    intent.putExtra(Settings.EXTRA_APP_PACKAGE, activity.getPackageName());
    activity.startActivity(intent);
}
```

Show this CTA when `onPermissionResult(granted = false, ...)` fires, OR when `getNotificationPermissionStatus()` returns `"denied"` and the customer wants to re-enable.

---

## Status check — `getNotificationPermissionStatus()`

```kotlin
when (PushEngage.getNotificationPermissionStatus()) {
    "granted" -> { /* notifications enabled */ }
    "denied"  -> { /* show the settings deep-link CTA */ }
}
```

Returns **only** `"granted"` / `"denied"` — there is **no** third state (no `"not_determined"`). If you've never asked, status is `"denied"` until the user grants.

This contrasts with iOS where `UNAuthorizationStatus` has `.notDetermined`. On Android, you can't distinguish "user hasn't been asked yet" from "user has been asked and denied" at this API level.

---

## Where this fits in the integration flow

After `init-code.md` (Application subclass init) and before verification. The orchestrator (`SKILL.md`) reads this file in Part 3d.

If the customer is using Jetpack Compose, also read `jetpack-compose.md` for Compose-specific call-site patterns and the `NotificationPermissionGate` snapshot caveat.
