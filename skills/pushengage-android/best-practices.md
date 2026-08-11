# Android PushEngage Best Practices (ANDROID-BP)

Android-specific rules. Cite by rule ID in audit findings and debugging output. Cross-platform rules live in `pushengage/best-practices.md` (`XP-BP-NN`).

---

## ANDROID-BP-01 — `requestNotificationPermission` requires `ComponentActivity`

**Rule:** The call site for `PushEngage.requestNotificationPermission(activity, callback)` must be a class that extends `androidx.activity.ComponentActivity` (or a subclass: `AppCompatActivity`, `FragmentActivity`). A class that extends plain `android.app.Activity` will not compile.

**Why:** The SDK's signature is `public static void requestNotificationPermission(ComponentActivity activity, PushEngagePermissionCallback callback)`. The compiler rejects a plain `Activity` argument at the call site. Customers occasionally hit this with legacy Activities or hand-rolled bases.

**How to apply:** Audit check (ANDROID-AUDIT-02) — read all `*Activity.kt` / `*Activity.java`, check the base class. If the class is `Activity` (not a `ComponentActivity` subclass) and it calls `requestNotificationPermission(this, ...)`, flag it. Suggested fix: change base to `AppCompatActivity` (most apps already pull in `appcompat`). See KI-A002.

## ANDROID-BP-02 — Declare `POST_NOTIFICATIONS` explicitly in the app Manifest

**Rule:** For apps targeting API 33+, add `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />` to `app/src/main/AndroidManifest.xml` explicitly, even though it is not strictly required for the prompt to appear.

**Why:** The PushEngage SDK's own library manifest declares this permission, and Android's manifest merger propagates library permissions into the consuming app — so the runtime prompt normally appears even without an app-level entry. The explicit entry is belt-and-braces: it documents intent, shows up in code review, and protects against the cases where the merged entry is missing (the app strips it with `tools:node="remove"`, or merger is otherwise blocked).

**How to apply:** Audit check (ANDROID-AUDIT-01) — read `app/src/main/AndroidManifest.xml`; grep for `POST_NOTIFICATIONS`. If absent from the app manifest, flag as **Worth fixing** (clarity), NOT as the likely cause of a missing permission dialog. Before blaming the manifest for "no dialog appears," verify the permission is actually absent from the MERGED manifest (`app/build/intermediates/merged_manifests/...`) or that a `tools:node="remove"` strips it. See KI-A001.

## ANDROID-BP-03 — Request permission at a meaningful moment

**Rule:** Do NOT call `requestNotificationPermission` from `Application.onCreate()` or `MainActivity.onCreate()`. Trigger it on a user action that makes the prompt make sense — e.g., after onboarding, when the user taps "Enable notifications," or after they complete a key first-use step.

**Why:** Android only shows the system permission dialog ONCE in most cases. If the user denies it (or dismisses it without granting) during cold boot before they've seen any value, the only path back is `Settings → Apps → <App> → Notifications`. This is the Android analog of XP-BP-03 (cross-platform "meaningful moment" rule).

**How to apply:** Audit check (ANDROID-AUDIT-06) — grep for `requestNotificationPermission` inside `onCreate` of any `Application` or `MainActivity` class. Flag with this rule ID and suggest moving to a button handler.

## ANDROID-BP-04 — `setSmallIconResource` requires a real drawable

**Rule:** When calling `PushEngage.setSmallIconResource("ic_xxx")`, a drawable named `ic_xxx` MUST exist in `res/drawable*/` (any density bucket). The icon should be a transparent white-on-transparent PNG or vector (mono-tone, Android system requirement).

**Why:** The SDK silently falls back to a generic bell icon if the named drawable doesn't resolve. No exception, no log. The customer wonders why their custom icon isn't showing up — and there's no error trail.

**How to apply:** Audit check (ANDROID-AUDIT-03) — for every call to `setSmallIconResource("X")`, verify `res/drawable*/X.{xml,png,webp}` exists. If not, flag with this rule ID. Mention the format requirement (white-on-transparent monochrome) when proposing the fix; don't silently invent a drawable.

## ANDROID-BP-05 — Create notification channels in `Application.onCreate()` on first launch

**Rule:** If the customer wants custom notification channels (beyond the SDK's auto-created default), create them in `Application.onCreate()` during first launch — not lazily on the first push, and not per-push.

**Why:** Channels created after API 26+ first reads them are silently ignored. Creating per-push is a no-op for any push after the first. The auto-default channel suffices for single-channel apps; multi-channel customers (e.g., "Promotions" vs. "Order updates") need this rule.

**How to apply:** Recommend during the `init-code.md` step if the customer mentions multi-channel needs. No automatic audit (the absence of custom channels is fine for most apps).

## ANDROID-BP-06 — Gate `enableLogging(true)` behind `BuildConfig.DEBUG`

**Rule:** Wrap `PushEngage.enableLogging(true)` in `if (BuildConfig.DEBUG) { ... }`. Never ship release builds with logging on.

**Why:** Verbose SDK logs may include subscriber IDs, FCM tokens, and request/response payloads. Release builds emitting these to logcat is a privacy and security smell. (This is the Android analog of XP-BP-07.)

**How to apply:** Audit check (ANDROID-AUDIT-07) — grep for `enableLogging(true)`; flag any unconditional call (i.e., not inside a `BuildConfig.DEBUG` guard). The fix is a one-line wrap.

## ANDROID-BP-07 — `google-services.json` belongs at `app/google-services.json`

**Rule:** The downloaded `google-services.json` file goes inside the **app module directory** (`app/google-services.json`), NOT at the project root.

**Why:** The `google-services` Gradle plugin looks for the file inside the module it's applied to. If placed at the project root, the build fails with: `File google-services.json is missing. The Google Services Plugin cannot function without it.` This is a common integration mistake — Firebase's download UX doesn't make the destination obvious.

**How to apply:** Audit check (ANDROID-AUDIT-04) — check whether `google-services.json` exists at the project root but NOT at `app/`. If so, flag with this rule ID and KI-A006; the fix is `mv google-services.json app/`.

## ANDROID-BP-08 — `google-services` plugin must apply AFTER `com.android.application`

**Rule:** In the app-level `build.gradle*` `plugins { ... }` block, list `id 'com.google.gms.google-services'` AFTER `id 'com.android.application'` (and after the Kotlin plugin if present).

**Why:** The Google Services plugin requires the Android application plugin to already be registered. Wrong order → build error "google-services plugin not applied" or "The Google Services Plugin cannot function without the Android Application plugin."

**How to apply:** Audit check (ANDROID-AUDIT-05) — read `app/build.gradle*`; verify ordering inside the `plugins { ... }` block. If `google-services` appears before `com.android.application`, flag with this rule ID and KI-A007.

## ANDROID-BP-09 — Java callers must pass ALL positional args to `TriggerCampaign` / `TriggerAlert` / `Goal`

**Rule:** Kotlin `data class`es without `@JvmOverloads` expose only the all-args constructor to **Java**; Kotlin callers may use the default params. Among the model classes, `TriggerCampaign(campaignName, eventName, referenceId, profileId, data)`, `TriggerAlert(...)`, and `Goal(name, count, value)` do NOT have `@JvmOverloads`, so Java code must pass all positional arguments — pass `null` explicitly for optional fields rather than omitting them. As of `0.1.0`, `TrackEvent` is the one model class WITH `@JvmOverloads` — `Goal`, `TriggerCampaign`, and `TriggerAlert` still lack it.

**Why:** Without `@JvmOverloads`, the generated Java constructor only exposes the full-arity form. Passing fewer args either fails to compile or — worse — silently binds the last positional arg to the wrong slot if the types happen to be compatible. `TriggerCampaign` requires 5 args; `TriggerAlert` requires 11; `Goal` requires 3. (Kotlin callers are unaffected — they can rely on the default params and pass only the required arguments.)

**How to apply:** Audit check (ANDROID-AUDIT-11) — grep for `new TriggerCampaign(` and count commas+1 against the required arity. Flag undersized calls with this rule ID. Note for `TriggerAlert`: `expiryTimestamp` is a `Date`, not a `String`.

## ANDROID-BP-10 — Provide a Settings deep-link helper for denial recovery

**Rule:** When the user denies the notification permission, the system silently blocks subsequent `requestNotificationPermission` calls. There is NO automatic second prompt. Surface a one-tap "Open notification settings" button using `Settings.ACTION_APP_NOTIFICATION_SETTINGS`.

**Why:** The SDK cannot re-prompt on its own. After repeated denial (typically the second, on Android 13+), the system permanently suppresses the dialog — the only path back is system settings. Customers who don't surface this end up with a permanently-disabled cohort that can never re-enable from inside the app.

**How to apply:** Recommend during `permission-flow.md`. The helper is ~5 lines:
```kotlin
fun openAppNotificationSettings(activity: Activity) {
    val intent = Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).apply {
        putExtra(Settings.EXTRA_APP_PACKAGE, activity.packageName)
    }
    activity.startActivity(intent)
}
```
No automatic audit; surface as a "worth fixing" nudge during integration review.

## ANDROID-BP-11 — Don't name a Compose composable the same as the `Application` subclass

**Rule:** In Jetpack Compose projects, do NOT name a top-level composable `MyApp()` if the customer also has `class MyApp : Application()`. Convention: name the composable `AppContent()`, and the Application class `MyApplication` or `PEApplication`.

**Why:** Kotlin compilation surfaces "ambiguous reference to MyApp" errors when both are in scope. The Compose codelab default of `@Composable fun MyApp()` collides with the customer-named Application subclass — common because PushEngage init goes in the Application class.

**How to apply:** Audit check (ANDROID-AUDIT-12) — if Compose is detected (grep `androidx.compose` in `app/build.gradle*`), look for `@Composable fun MyApp(` and `class MyApp : Application(` in the same module. If both present, flag with this rule ID and propose renaming the composable to `AppContent()`. Heuristic rule, not hard.

## ANDROID-BP-12 — `NotificationPermissionGate` Composables must not snapshot status via `remember{}`

**Rule:** When a Composable conditionally renders based on `PushEngage.getNotificationPermissionStatus()`, do NOT capture the value with bare `remember { PushEngage.getNotificationPermissionStatus() }`. Drive the state from a lifecycle observer (`LaunchedEffect` keyed on `Lifecycle.Event.ON_RESUME`) into a `mutableStateOf`.

**Why:** `remember {}` captures a snapshot at first composition. If the user grants or revokes the permission via system settings (mid-session), the Composable will not recompose, and the gate will show stale state. The user appears stuck in a denied UI even after they've granted.

**How to apply:** Recommend in `jetpack-compose.md`. Audit hint: grep for `remember { PushEngage.getNotificationPermissionStatus()` — if found, flag and propose the lifecycle-driven pattern.

## ANDROID-BP-13 — Document OEM battery-optimization caveat in the customer's support FAQ

**Rule:** Surface guidance about OEM-specific background restrictions (Xiaomi, OnePlus, Huawei, Samsung "deep sleep," etc.) in the customer's user-facing FAQ or in-app help. After a user force-stops the app on these OEMs, background services are aggressively restricted and PushEngage notifications stop arriving until the user reopens the app or disables battery optimization for it.

**Why:** This is NOT an SDK bug — it's a device-vendor restriction beyond PushEngage's control. The mitigation is customer-side: tell users about `Settings → Battery → Battery Optimization → Don't optimize`, and optionally surface an in-app prompt that deep-links there. (See KI-A003.)

**How to apply:** Recommend during integration wrap-up or when debugging "notifications work for a few days then stop" symptoms. No code-level audit (this is a docs/FAQ rule). Pair with the diagnostic-tree Android Branch 7 entry.

## ANDROID-BP-14 — `identify(...)` keys must be one of the 12 predefined subscriber fields

**Rule:** When calling `PushEngage.identify(JSONObject)` (added in `0.1.0`), every key MUST be one of the 12 predefined subscriber fields: `first_name`, `last_name`, `email`, `phone`, `gender`, `dob`, `language`, `profile_id`, `country`, `city`, `state`, `zip`. For arbitrary/custom attributes use `addSubscriberAttributes(JSONObject)` instead. Treat identify values (email/phone/etc.) as PII — pairs with XP-BP-05 / ANDROID-BP-06 (don't log them in release builds).

**Why:** An out-of-set key fails client-side with `onFailure(400, "Key <k> is not valid. Valid keys are …")` and **never reaches the server** — no network call. Customers expecting `identify` to behave like the web `addAttributes` (arbitrary keys) will be confused when nothing is sent. Values must also be `String`, `Number`, or `Boolean`; anything else → `onFailure(400, "Value must be a string, number, or boolean.")`.

**How to apply:** When a customer calls `identify(...)`, check the JSON keys against the 12-field list and the values against String/Number/Boolean. If a key is custom, recommend `addSubscriberAttributes(...)`. This is a runtime 400 (not a compile error), so surface it as a best-practice nudge rather than a static audit.

## ANDROID-BP-15 — Register `setFcmConfigErrorListener` in debug builds to catch FCM config mismatches

**Rule:** In debug builds, register `PushEngage.setFcmConfigErrorListener { code, msg -> … }` (added in `0.1.0`) — ideally **before** `Builder.build()` — to surface FCM configuration mismatches between the app's `google-services.json` and the dashboard service account. Marshal to the UI thread inside the callback before touching UI; gate any logging behind `BuildConfig.DEBUG` and treat the message as potentially containing config identifiers.

**Why:** A sender-id / project-id mismatch between `google-services.json` and the PushEngage dashboard's Firebase service account is the #1 silent "no pushes ever arrive" cause — previously invisible. The listener fires `PEErrorCodes` 5001 (`FCM_SENDER_ID_MISMATCH`), 5002 (`FCM_PROJECT_ID_MISMATCH`), 5003 (`FCM_LOCAL_CONFIG_INVALID`), or 5004 (`FCM_CONFIG_BOTH_MISMATCH`). `FcmConfigErrorListener` is a Kotlin `fun interface`, so a lambda / SAM works. Note the callback may fire on a background networking thread, a FIS background thread, or the caller thread, and the init-time advisory check can fire against stale cached preferences — debounce before taking destructive action.

**How to apply:** Recommend during `init-code.md` / debugging "no notifications at all" symptoms. Pass `null` to clear the listener. No automatic audit; surface as a "worth doing" nudge in debug integrations.

---

Spokes citing Android rules: just write the rule ID (e.g., "violates ANDROID-BP-01, MainActivity extends plain Activity"). Customer can click into this file for the why.
