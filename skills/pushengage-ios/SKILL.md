---
name: pushengage-ios
description: Use ONLY when the active repo is a native iOS app (Podfile, Package.swift, or *.xcodeproj present at the project root) AND there is NO `pubspec.yaml` with a `flutter:` SDK block AND NO `package.json` with `react-native` or `expo` dependency. Handles full PushEngage integration end-to-end (push notifications and in-app messaging), plus debugging existing integrations. Do not use for Android, Flutter, or React Native apps even if PushEngage is mentioned.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.1.0"
---

# PushEngage — iOS Spoke

You're working in an iOS app. This skill walks the customer through integrating PushEngage from zero to verified test push, or debugs a broken integration.

## Routing

| Customer intent | Read this file next |
|---|---|
| "Integrate PushEngage" / "Set up PushEngage" (greenfield or partial) | This file's "Integration flow" section below. |
| "PushEngage isn't working" / specific symptom | Hand off to `pushengage-debug/SKILL.md` (skill: `pushengage-debug`). |
| "Audit my PushEngage setup" | Read `audit-checks.md` and run the checks. |
| "Set up / add in-app messages", "trigger an in-app message", "in-app message never shows", custom action buttons | Read `in-app-messaging.md` (iOS API + setup), then `../pushengage/in-app-messaging.md` for the platform-independent semantics. Requires SDK `1.1.0`. |
| Concept / terminology question | Read `../pushengage/concepts.md`. |
| Best practices / rule lookup | Read `best-practices.md` (iOS-specific) or `../pushengage/best-practices.md` (cross-platform). |
| API surface / "does method X exist" | Read `version-matrix.md` first; if not listed, fetch only URLs from `references.md`. |

## Integration flow (greenfield)

This is the happy path. Follow the parts strictly in order — earlier artifacts feed later steps.

### Part 1 — Discovery & prereqs

Before touching anything, gather:

1. iOS deployment target (read project file / `Podfile` `platform :ios, 'X.Y'`).
2. Package manager: SPM only / CocoaPods only / both / neither (look for `Podfile`, `Package.swift`, `Pods/`, SPM `Package.resolved`).
3. UI framework: UIKit (find `AppDelegate.swift`) or SwiftUI (find `@main struct` and `@UIApplicationDelegateAdaptor`).
4. Existence of any other SDK that swizzles `UIApplicationDelegate` (search for `FirebaseApp.configure`, `OneSignal.initialize`, `Branch.getInstance`, etc.).
5. Whether the customer already has a PushEngage account + site → ask the customer.

Output a one-paragraph integration plan in chat. Wait for the customer to confirm before any edits.

### Part 2 — APNs + dashboard (out-of-code, before any code edits)

Counter-intuitive but right: do these BEFORE editing code. The dashboard's App ID is needed for `setAppID(id:)`; APNs cert generation has a real-world wait at Apple Developer portal.

1. Read `provider-setup.md`. Walk the customer through APNs cert generation. Capture: `.p12` (the file lives with the customer, not us), `.p12` password (NEVER ask the customer to paste this in chat), Bundle Identifier.
2. Read `dashboard-setup.md`. Walk the customer through Site Settings → Installation → iOS SDK. Capture: App ID UUID (ask the customer to paste this in chat).

Do not proceed to Part 3 until both steps confirm.

### Part 3 — In-code edits

#### 3a — Install

Read `install.md`. Choose SPM vs CocoaPods per the decision rule there.

- **Auto-edit + announce posture:** Podfile, Package.swift dep additions. Show the diff in chat, apply.
- Run `pod install` (CocoaPods path) and confirm success.

#### 3b — Main target init code

Read `init-code.md`. Pick the template by UI framework + integration mode:

- Default = swizzling. Switch to manual only on IOS-BP-02 trigger (Firebase, policy, etc.).
- Templates: Swift/UIKit, Swift/SwiftUI, Objective-C — swizzling and manual variants exist.

**Diff + confirm posture for AppDelegate edits.** Show the proposed diff. Ask the customer "apply?" before writing.

Insert the App ID UUID captured in Part 2.

Also:
- Enable Push Notifications capability — auto-edit + announce.
- Background Modes (Remote notifications, Background fetch): only if the customer will use silent/background pushes — see IOS-BP-03. Don't enable it by default for alert-only integrations.
- Insert Info.plist keys per `init-code.md` ("Info.plist additions" section).

#### 3c — Notification Service Extension (NSE)

Read `nse-setup.md`. Required for rich-media notifications and reliable analytics. Default: do this.

NSE target creation: instruct the customer via Xcode UI (don't try to edit `.pbxproj` programmatically). NSE source file: replace contents with the template (auto-edit + announce; it's a freshly generated file).

From `1.0.0`, the NSE target uses the **`PushEngageExtension`** pod/product and `import PushEngageExtension` (NOT `PushEngage`); the three NSE methods are called on the `PushEngageExtension` class.

App Groups setup is part of this — both main app and NSE target must share a group, and both Info.plists need `PushEngage_App_Group_Key`.

#### 3d — Notification Content Extension (NCE) — OPTIONAL

Read `nce-setup.md` only if the customer explicitly asks for custom expanded UI. Default: skip; mention it exists; ask if they want it. From `1.0.0`, the NCE target also uses the **`PushEngageExtension`** pod/product and `import PushEngageExtension`; `getCustomUIPayLoad(for:)` is called on the `PushEngageExtension` class.

#### 3e — In-App Messaging — OPTIONAL

Read `in-app-messaging.md` only if the customer asks for in-app messages (or already has IAM campaigns on the dashboard). Default: skip; mention it exists in one line ("the same SDK also shows dashboard-designed in-app messages with no extra setup — say so if you want them"). IAM needs nothing beyond Part 3b: no pod, no capability, no permission. Optional pieces are the app-wide `setIAMCustomActionHandler` registration in the AppDelegate (diff + confirm) and the background-refresh Info.plist key + Background fetch mode (both or neither). IAM verification is separate from Part 4 and does **not** need a real device.

### Part 4 — Verification

The success criterion is **a real test notification reaches the customer's device**.

1. Customer connects a real device (NOT simulator — APNs requires real device).
2. Customer launches the app once → grants notification permission when prompted.
3. Customer goes to PushEngage dashboard → Campaign → Push Broadcasts → Create New Push Broadcast → fills minimal fields → sends to the test device.
4. Ask the customer: "Did the notification arrive?"

**If yes** → go to Part 5.
**If no** → hand off to `pushengage-debug/SKILL.md` with the symptom "test push didn't arrive."

### Part 5 — Post-integration best-practices nudge

Single short message. Don't auto-apply; list rule IDs and one-line summaries:

- XP-BP-01: move the App ID UUID out of source (you currently have it hardcoded).
- XP-BP-02: consider a second PushEngage site for test/QA traffic.
- XP-BP-03: request notification permission at a meaningful user moment, not on launch.
- XP-BP-07: ensure `enableLogging` is gated by `#if DEBUG`.

(Customize the list to what you actually saw in the customer's code — don't dump all of them if some don't apply.)

## Debugging entry points

If the customer says any of the following — or if Part 4 verification fails — invoke `pushengage-debug/SKILL.md`:

- "Notifications aren't arriving"
- "Permission dialog never shows" / "always returns true immediately"
- "Deep link is broken"
- "Subscriber count looks wrong" / "I'm not registering as a subscriber"
- "App crashes on launch after install"
- "Build fails" (from `1.1.0`: also "pod install says PushEngage (1.1.0) does not support iOS 12/13/14" → KI-009 / IOS-AUDIT-18)
- "In-app message never appears" / "in-app button does nothing" / "custom action never reaches my code" → diagnostic tree Branch 8 + `in-app-messaging.md`

The debug skill handles audit + known-issues + symptom tree.

## What you must not do

- Don't invent PushEngage method names. If you're not sure a method exists, check `version-matrix.md`. If still unsure, fetch an allowlisted URL from `references.md`. If still unsure, say so.
- Don't auto-edit AppDelegate without showing the diff and getting confirmation (entry-point code is diff + confirm).
- Don't touch files outside the integration scope. No refactoring, no Swift-version migrations, no formatting passes.
- Don't fetch URLs not in `references.md`.
- Don't ask the customer for the `.p12` private key password in chat — they enter it directly on the dashboard.
- Don't reference internal IAM types (`IAMController`, `IAMPosition`, …) or invent IAM methods (`enableInAppMessaging`, `pauseInAppMessages`, `dismissCurrentMessage`) — the public IAM surface is exactly `triggerIAMEvent`, `setIAMCustomActionHandler`, `PEIAMCustomActionHandler`.
- Don't invent the dashboard click-path for creating IAM campaigns; the docs don't specify it.
