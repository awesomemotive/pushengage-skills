---
name: pushengage-android
description: Use ONLY when the active repo is a native Android app (build.gradle or build.gradle.kts at root with AndroidManifest.xml under app/src/main) AND there is NO `pubspec.yaml` with a `flutter:` SDK block AND NO `package.json` with `react-native` or `expo` dependency. Handles full PushEngage integration end-to-end, plus debugging existing integrations. Do not use for iOS, Flutter, or React Native apps even if PushEngage is mentioned.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.0.0"
---

# PushEngage — Android Spoke

You're working in an Android app. This skill walks the customer through integrating PushEngage from zero to verified test push, or debugs a broken integration.

## Routing

| Customer intent | Read this file next |
|---|---|
| "Integrate PushEngage" / "Set up PushEngage" (greenfield or partial) | This file's "Integration flow" section below. |
| "PushEngage isn't working" / specific symptom | Hand off to `pushengage-debug/SKILL.md` (skill: `pushengage-debug`). |
| "Audit my PushEngage setup" | Read `audit-checks.md` and run the checks. |
| Concept / terminology question | Read `../pushengage/concepts.md`. |
| Best practices / rule lookup | Read `best-practices.md` (Android-specific) or `../pushengage/best-practices.md` (cross-platform). |
| API surface / "does method X exist" | Read `version-matrix.md` first; if not listed, fetch only URLs from `references.md`. |

## Integration flow (greenfield)

This is the happy path. Follow the parts strictly in order — earlier artifacts feed later steps.

### Part 1 — Discovery & prereqs

Before touching anything, gather:

1. **SDK levels** — read `compileSdk`, `minSdkVersion`, `targetSdkVersion` from `app/build.gradle` or `app/build.gradle.kts`.
2. **Gradle DSL** — Groovy (`build.gradle`) vs Kotlin (`build.gradle.kts`). The install template differs.
3. **UI framework** — Compose vs classic Views. Grep `androidx.compose` in `app/build.gradle*`; if present, flag the Compose sub-flow (Part 3c).
4. **Existing Application subclass** — grep for `extends Application` / `: Application()` under `app/src/main/`. If one exists, init code extends it; if not, we create one and register it in `AndroidManifest.xml`.
5. **MainActivity base class** — plain `Activity` vs `AppCompatActivity` vs `ComponentActivity` (Compose). This affects which permission-flow snippet applies.
6. **Account state** — ask the customer:
   - Do you already have a Firebase project for this app?
   - Do you already have a PushEngage account + site for this app?
7. **Compose guidance** — if Compose was auto-detected in step 3, confirm with the customer ("I see Jetpack Compose in your build file — apply the Compose-specific snippets in Part 3c?").

Output a one-paragraph integration plan in chat. Wait for the customer to confirm before any edits.

### Part 2 — Firebase + dashboard (out-of-code, before any code edits)

Counter-intuitive but right: do these BEFORE editing code. The dashboard's App ID is needed for `setAppId(...)`; Firebase project + Service Account upload have real-world setup steps in the Firebase console.

1. Read `firebase-setup.md`. Walk the customer through creating (or reusing) the Firebase project, adding the Android app with the correct `applicationId`, downloading `google-services.json`, and generating a Service Account JSON. Capture:
   - `google-services.json` file path inside the customer's repo (typically `app/google-services.json`) — the file lives with the customer.
   - Service Account JSON file path on the customer's machine — **NEVER ask the customer to paste the JSON contents in chat.** They upload it directly to the PushEngage dashboard.
   - Sender ID and `applicationId` for cross-checking later.
2. Read `dashboard-setup.md`. Walk the customer through Site Settings → Installation → Android SDK and through Service Account JSON upload. Capture: **App ID UUID** (ask the customer to paste this in chat — it is a public identifier, not a secret).

Do not proceed to Part 3 until both steps confirm.

### Part 3 — In-code edits

#### 3a — Install

Read `install.md`. Pick the Groovy DSL or Kotlin DSL template per the detection from Part 1.

- **Auto-edit + announce posture for config files:** root `build.gradle*`, `app/build.gradle*`, `settings.gradle*`, `AndroidManifest.xml`, and placement of `google-services.json`. Show the diff in chat, then apply.
- Apply the Google Services plugin, Firebase BoM (if absent), and the PushEngage Android SDK dependency.
- Add the `<uses-permission>` entries to `AndroidManifest.xml`, including an explicit `POST_NOTIFICATIONS` entry if `targetSdkVersion >= 33`. (The SDK's own library manifest already declares `POST_NOTIFICATIONS` and manifest merger propagates it — the explicit app-level entry is for clarity, not correctness.)
- Run a Gradle sync (or instruct the customer to) and confirm success before moving on.

#### 3b — Application class init code

Read `init-code.md`. Pick the template by language (Kotlin vs Java) and by whether an `Application` subclass already exists.

- If no `Application` subclass exists: create one (e.g., `MyApp.kt`) and register it via `android:name=".MyApp"` in `AndroidManifest.xml` (auto-edit + announce for the manifest).
- If one exists: extend it — do not replace it.

**Diff + confirm posture for Application class edits.** Show the proposed diff. Ask the customer "apply?" before writing.

Insert the App ID UUID captured in Part 2 into the `setAppId(...)` call. Per XP-BP-01, leave a comment recommending the customer move it to `BuildConfig` / resources later.

#### 3c — Jetpack Compose deltas (Compose-only)

**Run this sub-step only if Compose was detected in Part 1.**

Read `jetpack-compose.md`. Apply Compose-specific deltas:

- Permission request via `rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission())` instead of `ActivityCompat.requestPermissions`.
- Where to invoke the PushEngage permission helper from a `Composable` (typically inside a `LaunchedEffect` or a button click handler).
- `ComponentActivity` is the expected base class — confirm against what was detected in Part 1.

Diff + confirm for any edits inside a Composable file the customer authored.

#### 3d — Runtime permission flow

Read `permission-flow.md`. Pick the snippet for the customer's MainActivity base class (`AppCompatActivity` / `ComponentActivity`) and `targetSdkVersion` (Android 13+ needs the runtime `POST_NOTIFICATIONS` request; below 13 the permission is granted at install time but you still call PushEngage's subscriber registration).

**Diff + confirm posture for MainActivity edits.**

Per XP-BP-03, prompt for permission at a meaningful moment, not immediately on `onCreate`. Suggest the right call site; do not force `onCreate`.

### Part 4 — Verification

The success criterion is **a real test notification reaches the customer's device**.

1. Customer connects a **real device with Google Play Services** — NOT an emulator without GMS. FCM will not deliver to a GMS-less image.
2. Customer installs and launches the app once → grants the notification permission when prompted (Android 13+).
3. Customer goes to PushEngage dashboard → Campaign → Push Broadcasts → Create New Push Broadcast → fills minimal fields → sends to the test device (or to "All Subscribers" if it's the only one).
4. Ask the customer: "Did the notification arrive?"

**If yes** → go to Part 5.
**If no** → hand off to `pushengage-debug/SKILL.md` with the symptom "test push didn't arrive."

### Part 5 — Post-integration best-practices nudge

Single short message. Don't auto-apply; list rule IDs and one-line summaries — pick 2–3 that are actually relevant to what you just integrated. Examples:

- XP-BP-01: move the App ID UUID out of source into `BuildConfig` or a resource (you currently have it hardcoded).
- XP-BP-03: request notification permission at a meaningful user moment, not on launch.
- XP-BP-07: gate `enableLogging` behind `BuildConfig.DEBUG`.
- ANDROID-BP-04: set a proper monochrome notification icon — Android renders a white/silhouette bell if you ship `ic_launcher` as the small icon.
- ANDROID-BP-05: create named notification channels at app start (Android 8+) so users can manage categories.
- ANDROID-BP-13: warn the customer about OEM background restrictions (Xiaomi, Oppo, Vivo, Huawei) and link them to the audit check.

(Customize the list to what you actually saw in the customer's code — don't dump all of them if some don't apply.)

## Debugging entry points

If the customer says any of the following — or if Part 4 verification fails — invoke `pushengage-debug/SKILL.md`:

- "Notifications aren't arriving"
- "Permission prompt never shows" / "I denied it once and now I can't re-prompt"
- "Notifications work for a while then stop" → strong OEM background-restriction signal (Xiaomi / Oppo / Vivo / Huawei / OnePlus)
- "Notification icon is wrong" / "shows a white square or a bell"
- "Build fails" (Gradle sync, manifest merger, duplicate class, etc.)
- "App force-closes on launch after install" (often an `Application` init order issue or missing `google-services.json`)
- "Subscriber count looks wrong" / "I'm not registering as a subscriber"
- "Deep link from notification doesn't open the right screen"

The debug skill handles audit + known-issues + symptom tree.

## What you must not do

- **Don't invent PushEngage method names.** If you're not sure a method exists on the Android SDK, check `version-matrix.md`. If still unsure, fetch an allowlisted URL from `references.md`. If still unsure, say so — don't guess.
- **Don't auto-edit the customer's `Application` subclass or `MainActivity` without showing the diff and getting confirmation.** Entry-point code is diff + confirm. Config files (`build.gradle*`, `AndroidManifest.xml`, `settings.gradle*`) are auto-edit + announce.
- **Don't touch files outside the integration scope.** No refactoring, no Kotlin/Java migrations, no Gradle version bumps that weren't required, no formatting passes.
- **Don't fetch URLs not in `references.md`.** The allowlist exists so the customer can trust where you got information.
- **Don't ask the customer for Service Account JSON contents in chat.** It is a secret. They upload the file directly to the PushEngage dashboard; you only need to know the local path and confirm the upload succeeded.
