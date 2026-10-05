# Android Static Audit Checks

Audit runs first in the debugging flow (`pushengage-debug` Phase 1). Each check below:

- States what to read.
- States what pattern to look for.
- Cites the rule ID violated.
- Categorizes the finding as **Likely cause** / **Worth fixing** / **Cosmetic**.

Findings format (for output to the customer):

```
[Likely cause]  ANDROID-BP-01 — requestNotificationPermission called from a plain Activity
   File: app/src/main/java/com/example/MainActivity.kt
   Effect: compile error — the SDK signature requires ComponentActivity.
   Fix: Change the base class to AppCompatActivity / FragmentActivity /
        ComponentActivity.
```

---

## Read targets

For each audit run, read (or glob the paths first):

- `app/src/main/AndroidManifest.xml`
- `app/build.gradle` and `app/build.gradle.kts`
- Top-level `build.gradle` and `build.gradle.kts`
- `settings.gradle` and `settings.gradle.kts`
- All `**/*Activity.kt`, `**/*Activity.java`
- All `**/*Application.kt`, `**/*Application.java` (any class extending `Application`)
- All `.kt`/`.java` under `app/src/main/` that reference `PushEngage.`
- `app/google-services.json` AND `google-services.json` at project root (existence check only — don't read contents)
- `app/src/main/res/drawable*/` listing (for ANDROID-AUDIT-03)
- All Compose-related files under `app/src/main/` (any file that imports `androidx.compose.*`) — for ANDROID-AUDIT-12

Don't read further than this. The audit is bounded.

---

## Check ANDROID-AUDIT-01 — Missing `POST_NOTIFICATIONS` in app Manifest

**Reference:** ANDROID-BP-02 / KI-A001.
**Read:** `app/src/main/AndroidManifest.xml`.
**Pattern:** absence of `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />` from the APP manifest.
**Category:** Worth fixing (clarity). The SDK's library manifest declares this permission and manifest merger propagates it, so the dialog normally appears anyway. Escalate to Likely cause ONLY if the permission is also absent from the merged manifest (`app/build/intermediates/merged_manifests/...`) or a `tools:node="remove"` strips it.
**Fix:** add the `<uses-permission ...>` line inside `<manifest>`, before `<application>`.

## Check ANDROID-AUDIT-02 — Plain `Activity` base for `requestNotificationPermission`

**Reference:** ANDROID-BP-01 / KI-A002.
**Read:** all `*Activity.kt`/`*Activity.java`.
**Pattern:** a class whose declared base is plain `android.app.Activity` (Kotlin: `: Activity()`; Java: `extends Activity`) AND somewhere in the same module a call to `PushEngage.requestNotificationPermission(this, ...)` where `this` could resolve to that activity.
**Category:** Likely cause (compile error: `requestNotificationPermission(ComponentActivity, ...) cannot be applied to (...)`).
**Fix:** change the base class to `AppCompatActivity`, `FragmentActivity`, or `androidx.activity.ComponentActivity`.

## Check ANDROID-AUDIT-03 — `setSmallIconResource` references missing drawable

**Reference:** ANDROID-BP-04.
**Read:** all `.kt`/`.java` referencing `setSmallIconResource(`. Pull out the string literal argument. List `app/src/main/res/drawable*/`.
**Pattern:** `setSmallIconResource("X")` where no file named `X.xml` or `X.png` exists under any `res/drawable*/` folder.
**Category:** Likely cause (silent fallback to default bell icon; no exception, no log).
**Fix:** either add a drawable at `res/drawable/<X>.xml` (or `.png`) — transparent white-on-transparent monochrome — OR change the argument to a name that exists.

## Check ANDROID-AUDIT-04 — `google-services.json` at project root instead of `app/`

**Reference:** ANDROID-BP-07 / KI-A006.
**Read:** existence of `google-services.json` at project root AND `app/google-services.json`.
**Pattern:** `google-services.json` exists at project root but NOT at `app/google-services.json`.
**Category:** Likely cause (build fails with `File google-services.json is missing. The Google Services Plugin cannot function without it.`).
**Fix:** move the file from project root to `app/google-services.json`.

## Check ANDROID-AUDIT-05 — `google-services` plugin applied before `com.android.application`

**Reference:** ANDROID-BP-08 / KI-A007.
**Read:** `app/build.gradle` or `app/build.gradle.kts` `plugins { ... }` block.
**Pattern:** the line `id 'com.google.gms.google-services'` (Groovy) or `id("com.google.gms.google-services")` (Kotlin DSL) appears **before** `id 'com.android.application'` / `id("com.android.application")`.
**Category:** Likely cause (build fails with `google-services plugin not applied`).
**Fix:** reorder so `com.google.gms.google-services` comes AFTER `com.android.application`.

## Check ANDROID-AUDIT-06 — `requestNotificationPermission` in onCreate

**Reference:** ANDROID-BP-03 (mirrors XP-BP-03).
**Read:** all `*Activity.kt`/`*Activity.java`, all `*Application.kt`/`*Application.java`.
**Pattern:** `PushEngage.requestNotificationPermission(` appears inside an `onCreate(` function body (either Activity's `onCreate` or Application's `onCreate`).

Note: `Application.onCreate` won't compile (no Activity context) — but customers sometimes write this with a workaround that crashes at runtime. Flag both cases.

**Category:** Worth fixing (Activity.onCreate path: works but users typically deny when asked before seeing value).
**Fix:** move the call behind a user-driven moment (e.g. an "Enable notifications" button after onboarding).

## Check ANDROID-AUDIT-07 — `enableLogging(true)` unconditional

**Reference:** ANDROID-BP-06 / XP-BP-07.
**Read:** all `.kt`/`.java` referencing `enableLogging(`.
**Pattern:** `PushEngage.enableLogging(true)` NOT inside an `if (BuildConfig.DEBUG) { ... }` block (Kotlin) or `if (BuildConfig.DEBUG)` guard (Java).
**Category:** Worth fixing (leaks verbose logs to production builds).
**Fix:** wrap in `if (BuildConfig.DEBUG) { ... }`.

## Check ANDROID-AUDIT-08 — Loose SDK version pin

**Reference:** XP-BP-06.
**Read:** `app/build.gradle*` `dependencies { ... }` block.
**Pattern:** `pushengage-android-sdk` dependency uses any of:
- A range operator (`+`, `latest.release`, `1.+`).
- A property reference that resolves to a range.
- No version at all.

**Category:** Worth fixing.
**Fix:** pin exactly to the version in `version-matrix.md` (currently `1.0.0`).

## Check ANDROID-AUDIT-09 — App ID UUID hardcoded

**Reference:** XP-BP-01.
**Read:** all `.kt`/`.java` referencing `.setAppId(`.
**Pattern:** `.setAppId("<uuid-shaped-literal>")` where the literal matches `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}`.
**Category:** Cosmetic (works but bad hygiene; mixes envs / leaks via source control).
**Fix:** move to `BuildConfig` field per `buildType` / `flavor`, or to a `strings.xml` resource per environment.

## Check ANDROID-AUDIT-10 — `Application` subclass not registered in Manifest

**Reference:** ANDROID-BP (init never runs without manifest registration).
**Read:** all `*Application.kt`/`*Application.java` (any class extending `android.app.Application`); AND `app/src/main/AndroidManifest.xml`.
**Pattern:** an `Application` subclass exists in source AND `<application android:name="..." ...>` in the manifest is either absent or points to a different class name.
**Category:** Likely cause (PushEngage init code never executes; SDK silently never initializes).
**Fix:** add `android:name=".PEApplication"` (or whichever class) to the `<application>` element.

## Check ANDROID-AUDIT-11 — Java `TriggerCampaign` / `TriggerAlert` with missing positional args

**Reference:** ANDROID-BP-09.
**Read:** all `.java` files referencing `new TriggerCampaign(` or `new TriggerAlert(`.
**Pattern:**
- `new TriggerCampaign(` followed by fewer than 5 comma-separated args before the closing `)`.
- `new TriggerAlert(` followed by fewer than the required positional arg count.

Heuristic: count top-level commas inside the constructor call (skip commas inside nested parens/braces/string literals).

Note: `new TrackEvent(...)` (added in `0.1.0`) is **exempt** — `TrackEvent` ships with `@JvmOverloads`, so Java may pass 1-5 args. Do NOT flag undersized `new TrackEvent(` calls. Only `TriggerCampaign`, `TriggerAlert`, and `Goal` lack `@JvmOverloads`.

**Category:** Likely cause (compile error, or worse — type system binds last arg to wrong slot).
**Fix:** pass all positional args explicitly. Use `null` for optional fields. Reference `version-matrix.md` for the exact arity.

## Check ANDROID-AUDIT-12 — Compose composable named the same as `Application` class

**Reference:** ANDROID-BP-11.
**Read:** all `.kt` files importing `androidx.compose.*`. Pull all `@Composable fun <Name>()` declarations. Pull all `class <Name> : Application()` declarations across the module.
**Pattern:** the same name appears in both lists in the same module (e.g. `@Composable fun MyApp()` plus `class MyApp : Application()`).
**Category:** Likely cause (compile error: `ambiguous reference to MyApp` in Compose call sites).
**Fix:** rename the composable to `AppContent()` (convention). Don't rename the Application class — manifest references it.

This check is a **heuristic** — collisions can also arise across different packages with `import` shadowing; the check above only catches same-module collisions.

## Check ANDROID-AUDIT-13 — IAM custom-action handler registered only in an Activity

**Reference:** ANDROID-BP-16.
**Read:** all `.kt`/`.java` referencing `setIAMCustomActionHandler(`.
**Pattern:** every call site is inside an `Activity` subclass (or a Fragment / Composable) and none is inside the `Application` subclass.
**Category:** Worth fixing (custom buttons on other screens log `No custom action handler set` and never reach the app).
**Fix:** move the registration to `Application.onCreate()` (before or after `Builder.build()`), route on `parameters["action"]` there (not `actionId`). Quote `in-app-messaging.md`.

## Check ANDROID-AUDIT-14 — Internal IAM API or unnecessary forwarding

**Reference:** ANDROID-BP-17.
**Read:** all `.kt`/`.java` under `app/src/main/`.
**Pattern:** an import of `com.pushengage.pushengage.iam.` that is NOT `com.pushengage.pushengage.iam.action.IAMCustomActionHandler` (e.g. `IAMConfigurationManager`, `IAMController`), OR an `onRequestPermissionsResult` / `onConfigurationChanged` override that forwards to a `PushEngage`/IAM class.
**Category:** Worth fixing (restricted API — lint warning now, breakage on any SDK update; the forwarding is a no-op at best).
**Fix:** delete the reference/forwarding. The public IAM surface is `triggerIAMEvent`, `setIAMCustomActionHandler`, `IAMCustomActionHandler`.

## Check ANDROID-AUDIT-15 — Kotlin lambda passed to `setIAMCustomActionHandler`

**Reference:** ANDROID-BP-18 / KI-A008.
**Read:** all `.kt` files referencing `setIAMCustomActionHandler`.
**Pattern:** `PushEngage.setIAMCustomActionHandler {` or `setIAMCustomActionHandler({` (trailing lambda / lambda argument) in Kotlin.
**Category:** Likely cause (compile error — `IAMCustomActionHandler` is not a `fun interface`).
**Fix:** replace with `object : IAMCustomActionHandler { override fun onCustomAction(actionId: String, parameters: Map<String, Any>) { … } }`. Java lambdas are unaffected.

---

## Output formatting (when invoked from debug flow)

Group findings by category:

```
**Likely cause** (X findings):
1. [ANDROID-BP-02] Missing POST_NOTIFICATIONS — app/src/main/AndroidManifest.xml
2. [ANDROID-BP-01] Plain Activity base — app/src/main/java/com/example/MainActivity.kt:12

**Worth fixing** (X findings):
1. [ANDROID-BP-06] enableLogging unconditional — app/src/main/java/com/example/PEApplication.kt:18

**Cosmetic** (X findings):
1. [XP-BP-01] App ID hardcoded — app/src/main/java/com/example/PEApplication.kt:15
```

If 0 findings: report "No Android audit issues found." and hand off to debug Phase 2 (known-issues match).
