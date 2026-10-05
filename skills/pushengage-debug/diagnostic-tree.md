# Diagnostic Decision Tree

Used in Phase 3 of the debug skill (after audit and known-issues didn't resolve). Ask the customer to pick the closest symptom from the top-level branches; follow the tree.

## Top-level branches

1. **No notifications arrive at all.**
2. **Some notifications arrive; some don't.**
3. **Notifications arrive but the deep link / open behavior is wrong.**
4. **Permission flow is broken (dialog never shows, or returns immediately).**
5. **Subscriber count is wrong / device not registered.**
6. **App crashes / build fails.**
7. **(Android only) Notifications work for a while, then stop.**
8. **In-app message never appears / button does nothing / custom action never reaches the app.**

---

## Branch 1 — No notifications arrive

### 1.1 — Is the customer testing on a real device, not a simulator?

- **No → use a real device.** Stop. APNs requires a real device. (Try this first; resolves ~30% of "no notification" reports.)
- **Yes → 1.2**

### 1.2 — Is there a subscriber in the PushEngage dashboard for this device?

Ask the customer to log in to PushEngage dashboard → Subscribers → search by recent activity. (Filter by date if needed.)

- **Subscriber visible → 1.3**
- **No subscriber → 1.4**

### 1.3 — Subscriber exists; notification still doesn't arrive

Ask the customer to send a test notification from PushEngage dashboard (Campaign → Push Broadcasts → Create New Push Broadcast → target the test device).

- **Dashboard says "Delivered to APNs" → 1.3.1**
- **Dashboard says "Failed" or shows an error → 1.3.2**

#### 1.3.1 — APNs accepted; device shows nothing

Almost always a device-side filter:

- Focus mode / Do Not Disturb is on for the app.
- Notification settings disabled for the app (Settings → Notifications → <App>).
- Time-Sensitive Notifications toggle.
- App was force-quit and iOS is suppressing.

Ask customer to check each.

#### 1.3.2 — APNs rejected the notification

Likely root causes:
- `.p12` cert wrong (uploaded sandbox-only cert when production token; ours covers both, but check).
- Cert expired (Apple certs have 1-year lifespan).
- Bundle ID mismatch between cert, App Push Id, and the app build.
- Cert revoked.

Ask the customer to:
1. Re-check Site Settings → Installation → iOS SDK → App Push Id matches their actual Bundle ID.
2. Re-upload `.p12` (regenerate if expired).

### 1.4 — No subscriber in dashboard

Device never registered. Check in order:

- **1.4.1 — `setAppId` UUID:** verify it's the App ID UUID from the dashboard, not the App Push Id (Bundle ID) or some other string.
- **1.4.2 — Permission denied:** check Settings → Notifications → <App>. If never prompted, ensure `requestNotificationPermission` runs at some point.
- **1.4.3 — APNs registration failing:** enable logs (`log-readers.md`); the SDK doesn't print the APNs token itself — look for the `ApplicationService: <error>` registration-failure line, or for the absence of the `subscriber/add` request/response logs. Failure/absence = APNs unreachable (network) or capability misconfigured.
- **1.4.4 — Manual swizzling forwards missing:** hand back to iOS-spoke `audit-checks.md` IOS-AUDIT-04.

### 1.5 — Android-specific sub-branch (no notifications arrive)

If the customer is on Android, run these checks in order before falling through to the iOS-style branches above.

#### 1.5.1 — Real device with Google Play Services?

- Bare Android emulators (AOSP / non-Google-API images) lack Google Play Services → FCM cannot register a token → no subscriber, no notifications.
- Ask the customer to confirm they're testing on a real device, OR an emulator image with the **Google APIs** / Google Play target.
- On the device: Settings → Apps → Google Play services → should be present and up-to-date.

#### 1.5.2 — FCM token registration in verbose logs

Ask the customer to call `PushEngage.enableLogging(true)` in `Application.onCreate()` (gated by `BuildConfig.DEBUG`) and reproduce. See `log-readers.md` → Android section.

- Run `adb logcat -s PushEngage` and look for the FCM token line.
- **No FCM token line** → device never registered with FCM. Likely causes: no Play Services (see 1.5.1), wrong `google-services.json` (see 1.5.4), network blocked.
- **FCM token present but no "Subscriber registered" line** → token captured but `/subscribe` call failed. Re-check App ID UUID and Sender ID (1.5.3).

#### 1.5.3 — Sender ID match between Firebase Console and PushEngage dashboard

- In Firebase Console → Project settings → Cloud Messaging → note the **Sender ID** (numeric).
- In PushEngage dashboard → Site Settings → Installation → Android SDK → note the configured Sender ID / FCM Server Key fingerprint.
- They MUST match. Mismatch → FCM registers the device against a different project; PushEngage's server can't push to it.

#### 1.5.4 — `google-services.json` location and project match

- Must live at `app/google-services.json` (sibling of `app/build.gradle`), NOT at the project root (KI-A006).
- The `package_name` inside the JSON must match the app's `applicationId` in `app/build.gradle`. Mismatch → Gradle silently uses a default and FCM registration goes to the wrong project.

### 1.6 — Flutter-specific sub-branch (no notifications arrive)

If the customer is on a Flutter project (detected via `pubspec.yaml` with a `flutter:` SDK block), run these checks in order. The 1.5 Android-native sub-branch and 1.2/1.3 iOS-native branches still apply to whichever platform is failing — these add Flutter-specific layers on top.

#### 1.6.1 — Both `flutter pub get` AND `pod install` ran cleanly

The single most common Flutter integration mistake is running `flutter pub get` but forgetting `cd ios && pod install` afterwards (KI-F005). Ask the customer to re-run both:

```bash
flutter pub get
cd ios && pod install
```

If `pod install` errors out → see KI-F005 recovery steps (`pod repo update`, `pod deintegrate`).

#### 1.6.2 — `setAppId` runs in `main()` before `runApp`

Read `lib/main.dart`. The call must be:

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  PushEngage.setAppId(...);
  runApp(...);
}
```

NOT inside a widget's `initState` (FLUTTER-AUDIT-02 / FLUTTER-BP-01). Late init → cold-boot subscribers lost.

#### 1.6.3 — App ID UUID matches the platform

If the customer ships to both stores, verify `setAppId` uses platform-conditional branching (FLUTTER-AUDIT-07 / KI-F006):

```dart
PushEngage.setAppId(Platform.isIOS ? IOS_UUID : ANDROID_UUID);
```

A hardcoded single UUID will fail on one platform.

#### 1.6.4 — Android: POST_NOTIFICATIONS + google-services.json + plugin order

Falls through to 1.5. Specifically check: `POST_NOTIFICATIONS` in `android/app/src/main/AndroidManifest.xml` (FLUTTER-AUDIT-05 / KI-F004); `google-services.json` at `android/app/` not root (FLUTTER-AUDIT-06); google-services plugin applied after `com.android.application`.

#### 1.6.5 — iOS: NSE present + App Groups consistent

Falls through to 1.3. Specifically check: NSE target exists and builds without `APPLICATION_EXTENSION_API_ONLY` errors (FLUTTER-AUDIT-04 / KI-F002); both main and NSE targets have App Groups capability with the same group ID; both `Info.plist`s have identical `PushEngage_App_Group_Key` (FLUTTER-AUDIT-09 / KI-F007).

#### 1.6.6 — Logs

If audit passes: `if (kDebugMode) PushEngage.enableLogging(true);` and read `log-readers.md` → Flutter section. Method-channel markers (`PushEngage#<method>`) show which Dart calls reached native. Fall through to Branch 7 if Android.

### 1.7 — React Native-specific sub-branch (no notifications arrive)

For RN projects (`package.json` with `react-native` dep). The 1.5 Android and 1.3 iOS branches still apply per platform — these add RN-specific layers.

#### 1.7.1 — `npm install` + `cd ios && pod install` ran cleanly

KI-RN005. Re-run both. If `pod install` errors → `pod repo update && pod install`, then `pod deintegrate && pod install`.

#### 1.7.2 — Import has `@pushengage/` scope; `setAppId` in `index.js`; cross-platform App ID

Grep for `from 'pushengage-react-native'` (RN-AUDIT-01 / KI-RN001 — should be `@pushengage/pushengage-react-native`). Verify `setAppId` is at `index.js` top-level before `AppRegistry.registerComponent` (RN-AUDIT-02 / RN-BP-01). If both `ios/` and `android/` present, verify `Platform.OS === 'ios' ? IOS_UUID : ANDROID_UUID` branching (RN-AUDIT-07 / KI-RN006).

#### 1.7.3 — RN/SDK version compatibility

If `package.json` has `react-native: <0.78.0` AND `@pushengage/pushengage-react-native: >=0.0.2` → RN-AUDIT-14 / KI-RN009. Upgrade RN to 0.78+ (the only pre-floor release, `0.0.1`, predates most of the modern API — see KI-RN009).

#### 1.7.4 — Android: POST_NOTIFICATIONS + `google-services` plugin applied + google-services.json

Falls through to 1.5. RN-specific layers: `POST_NOTIFICATIONS` in Manifest (RN-AUDIT-05 / KI-RN004); `apply plugin: "com.google.gms.google-services"` in `android/app/build.gradle` (RN-AUDIT-15 / KI-RN010); `google-services.json` at `android/app/` (RN-AUDIT-06).

#### 1.7.5 — iOS: NSE present + App Groups + AppDelegate.mm complete

Falls through to 1.3. RN-specific layers: NSE builds without `APPLICATION_EXTENSION_API_ONLY` errors (RN-AUDIT-04 / KI-RN002); `AppDelegate.mm` has `swizzleInjectionWithIsEnabled:YES` in `-init` AND `setInitialInfoFor:with:` in `didFinishLaunchingWithOptions` (RN-AUDIT-08); App Groups + matching `PushEngage_App_Group_Key` (RN-AUDIT-09 / KI-RN007).

#### 1.7.6 — Logs

If audit passes: `if (__DEV__) PushEngage.enableLogging(true);` in `index.js`. Read `log-readers.md` → RN section.

---

## Branch 2 — Some arrive, some don't

Almost always either:
- Notification payload size > 4KB (APNs hard limit; check the campaign's image size).
- Silent (`content-available: 1`) notifications throttled by iOS (Apple rate-limits to ~2-3 per hour).
- One of: content delivery network issue, dashboard targeting filtering some devices out.

Ask the customer for two examples: one that worked, one that didn't. Compare the dashboard campaigns side by side.

---

## Branch 3 — Notification arrives but deep link / open is wrong

### 3.1 — URL is stripped to `scheme:///`

Capture the raw `userInfo` from a failing notification and compare the campaign's `pe.dl` vs `pe.u` fields — a scheme-only `pe.dl` is a campaign/dashboard config to fix.

### 3.2 — URL doesn't open at all

- Check `PushEngageAutoHandleDeeplinkURL` in Info.plist (KI-004 / IOS-BP-11).
- Check that `setNotificationOpenHandler` is registered (or that the flag is YES).

### 3.3 — URL opens but wrong screen

This is the customer's deep-link routing code. Hand the customer the `result.notificationAction.actionID` they're getting and ask them to map it to their screen routing.

### 3.4 — (Flutter) Cold-boot deep link lost; warm taps work

KI-F003 / FLUTTER-BP-02 / FLUTTER-AUDIT-12. `deepLinkStream.listen(...)` registered too late (downstream route's `initState`, post-auth, post-splash). Grep `lib/` for `deepLinkStream.listen`; if not in the root widget's `initState`, move it. Quote `pushengage-flutter/init-code.md` Section 1.

### 3.5 — (React Native) Cold-boot deep link lost; warm taps work

Same shape as 3.4. RN-AUDIT-12 / KI-RN003 / RN-BP-02. `onValueChanged(...)` registered too late (downstream screen's `useEffect`, post-auth, etc.). Grep `src/`, `App.tsx`, `index.tsx` for `onValueChanged`. Move to root component's `useEffect` with empty deps `[]`, return `() => subscription.remove()` (RN-BP-08). Quote `pushengage-react-native/init-code.md` Section 1.

---

## Branch 4 — Permission flow broken

### 4.1 — Dialog never shows

- `requestNotificationPermission` not called → call it.
- Permission already denied → recoverable only via Settings → Notifications → <App>. Surface a settings deep link in the app.

### 4.2 — Returns `true` immediately without showing dialog

On a reinstall where permission was previously determined, confirm the real state with `UNUserNotificationCenter.getNotificationSettings` rather than relying on the permission-request callback alone.

---

## Branch 5 — Subscriber not registered

Same as Branch 1.4.

---

## Branch 6 — Build / runtime crash

### 6.1 — Build error: `User Script Sandboxing`

IOS-BP-13. Build Settings → User Script Sandboxing → No.

### 6.2 — Build error: `Application extensions are not allowed to import …`

IOS-BP-12. Missing post-install hook in Podfile. Add it.

### 6.3 — Build error: `setIntialInfo` not found

IOS-BP-06. Fix the typo.

### 6.4 — Build error: `enableLogs` not found

IOS-BP-07. Use `enableLogging`.

### 6.5 — Runtime crash on launch

Almost always an init-order issue (IOS-BP-01) or a missing capability. Read the crash log; common signature:
- `EXC_BAD_ACCESS` inside `setInitialInfo` → `launchOptions` was nil unexpectedly; usually fine, but check.
- `NSInvalidArgumentException` referencing PushEngage → method called before init.

### 6.6 — (Android) Build error: `File google-services.json is missing`

→ KI-A006. `google-services.json` is at the project root instead of inside the app module. Move it to `app/google-services.json` (sibling of `app/build.gradle`) and re-sync Gradle.

### 6.7 — (Android) Build error: `google-services plugin not applied`

→ KI-A007. In `app/build.gradle(.kts)`, ensure `id 'com.google.gms.google-services'` is applied **after** `id 'com.android.application'`. Verify the project-level Gradle has the classpath/plugin declaration as well.

### 6.8 — (Android) Compile error on `requestNotificationPermission`

Customer sees a Java/Kotlin error like `requestNotificationPermission cannot be applied to ...` or "required `ComponentActivity`". → KI-A002. Calling Activity must extend `AppCompatActivity` / `FragmentActivity` / `ComponentActivity` — a plain `Activity` won't compile.

---

## Branch 7 — (Android only) Notifications work for a while, then stop

Customer reports notifications worked fine after install but stopped after the app sat in background for hours or days.

### 7.1 — Confirm device vendor

Ask for device make/model. The usual suspects: **Xiaomi/Redmi (MIUI), OnePlus (OxygenOS), Huawei/Honor (EMUI/HarmonyOS), Samsung (One UI, sometimes), Oppo/Vivo (ColorOS/FuntouchOS)**.

### 7.2 — OEM background restriction

→ KI-A003. The OEM's battery-optimization / aggressive app-kill is suppressing FCM background delivery. Not an SDK bug; no SDK-side fix.

**Workaround (customer must surface in their own app):**
- Show in-app advice: "For reliable notifications on <vendor> devices, disable battery optimization for this app."
- Deep-link to the system Settings page: `Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (or vendor-specific autostart page where available).
- Document in the customer's own FAQ.

### 7.3 — Confirm it's the OEM (not a real bug)

Ask the customer: does the notification arrive immediately if they manually open the app first, then send the test push? If yes → confirms OEM background kill. If no → fall back through Branch 1 (Android sub-branch 1.5).

### 7.4 — Flutter / RN apps: same mitigation, customer-side

Flutter and React Native apps hit the same OEM restriction. Same 7.2 pattern (in-app advice + Settings deep-link). Flutter: use `app_settings` / `permission_handler` package or a custom platform-channel method to launch `Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`. RN: use `Linking.openSettings()` (opens app notification settings; for battery-optimization specifically, use a native module or package like `react-native-battery-optimization-check`). Neither SDK can fix this — it's customer-app-side.

---

## Branch 8 — In-app messaging (all platforms)

IAM is a separate channel from push: it needs only the App ID, so none of Branch 1's subscriber/APNs/FCM checks apply unless the campaign targets *subscriber data*. Read `../pushengage/in-app-messaging.md` for the semantics before branching. Ask the customer to enable SDK logging first — every rejection is logged with the campaign id.

### 8.1 — Does the installed SDK have IAM at all?

- iOS `< 1.1.0`, Android `< 1.0.0`, Flutter `< 1.1.0`, RN `< 1.1.0` → no IAM API. Upgrade per the spoke's `version-matrix.md`. Wrapper apps: check the resolved native versions in `Podfile.lock` / Gradle, not just the plugin pin.
- Symptom "`triggerIAMEvent` is not a function / no such method / no member" → this is the answer.

### 8.2 — Message never appears

Work down in order:

1. **Synced?** Campaigns sync on init and ~every 6 h. Force-quit and relaunch after creating/editing a campaign. iOS: sync errors are error-level `os_log`, not stdout — capture both (`log-readers.md`).
2. **IAM enabled for the site, campaign published and active?** A disabled site purges local campaigns. (No MCP tool exposes this today — ask the customer.)
3. **Trigger:** app-open campaigns fire **once per process** — backgrounding is not a relaunch. Custom campaigns need the exact event string (case-sensitive); a blank name is rejected with `400 Event name is required`. Success from `triggerIAMEvent` only means "accepted".
4. **Date window** (UTC, inclusive) — expired or not yet started campaigns are skipped.
5. **Frequency cap** — a One-time campaign that already displayed on this device never shows again; clear app data / reinstall to reset.
6. **Audience** — device-data rules (`platform` is exactly `iOS`/`Android`, `language` is bare `en`, versions compare component-wise) always evaluate; **subscriber-data rules are skipped until the device is a push subscriber and the record was fetched once** (fresh install: usually the second open). Simulator / no-push-entitlement builds never satisfy them.
7. **Another message on screen?** One at a time; the next shows after dismissal. iOS queues at most 10.
8. **Trigger conditions typed wrongly?** A `string`-typed `greater than` never matches; numeric comparisons need a `number` condition. Malformed campaigns fail closed — the log says why.
9. iOS only: `CoreData model not found in resource bundle` in the log → KI-011 (manual framework integration). `BGTaskSchedulerPermittedIdentifiers` error line → KI-010, harmless.

If the log shows `Eligible message: <id>` (Android) / the iOS equivalent but nothing is visible → the host UI is covering it (Android: a full-screen `Dialog`/overlay added after the SDK's view; iOS: the SDK uses its own window above the app, so this is rare) — escalate with the log.

### 8.3 — Button does nothing

- **Action type spelling** on the dashboard must be exactly `open_url`, `request_notification_permission`, `dismiss`, or `custom` (lowercase). Anything else is inert: no click, message stays. Only that button is affected.
- **`open_url` with a non-http(s) URL** (custom scheme) is refused; the message still closes (KI-A009). Use a `custom` action.
- **`custom` with no handler/listener** → click recorded, message closes, app never notified → 8.4.
- The HTML must call `PEBridge.handleAction('<key>')` with a key that exists in the campaign's actions map; an unknown key leaves the message up.

### 8.4 — Custom action never reaches the app

- iOS: `setIAMCustomActionHandler` registered? Single slot — a later registration (e.g. in a view controller) replaced the AppDelegate one (IOS-AUDIT-17).
- Android: registered app-wide in `Application.onCreate()`? Per-Activity registration misses other screens (ANDROID-AUDIT-13). Kotlin lambda won't compile (KI-A008).
- Flutter: `onIAMCustomAction.listen` active in a live widget (FLUTTER-AUDIT-14)? Not cancelled by a popped route? Host `AppDelegate.swift` must NOT call `setIAMCustomActionHandler` (it overwrites the plugin's).
- RN: `onIAMCustomAction(cb)` subscribed in a mounted component (RN-AUDIT-19)? No `setIAMCustomActionHandlerEnabled` call (KI-RN015)? Host native code must NOT call `setIAMCustomActionHandler`.
- Handler fires but the `switch` never matches? It is switching on `actionId` (an internal dashboard key like `t04B05`) instead of `parameters["action"]` (the Action name). Log both and compare with the dashboard.
- Expecting it on the push listener (`setNotificationOpenHandler` / `deepLinkStream` / `onValueChanged`)? IAM never arrives there (XP-BP-09).

### 8.5 — Message can't be closed / shows too often / wrong order

- Unclosable: campaign lacks a Dismiss button and auto-dismiss, tap-outside off → dashboard authoring fix (XP-BP-11). The SDK draws no close button.
- Shows every launch: no frequency cap or Recurring with a short interval → dashboard fix.
- Wrong order: equal/unset priorities — give campaigns distinct priorities; lower number first.

---

**Tree termination:** if you reach a leaf and the customer's issue still isn't resolved, hand off to Phase 4 (support bundle).
