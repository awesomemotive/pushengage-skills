---
name: pushengage
description: Use when integrating, debugging, asking about, or auditing PushEngage push notifications or in-app messaging in any mobile app (iOS, Android, Flutter, React Native). Detects the customer's platform and hands off to the matching platform spoke.
license: MIT
compatibility: Works with any agent that supports the Agent Skills (SKILL.md) standard, including Claude Code and Cowork. Needs filesystem read access and a shell to detect the project type; some steps fetch allowlisted documentation URLs.
metadata:
  author: PushEngage
  version: "1.1.0"
---

# PushEngage — Entry Skill

This skill is the entry point for any PushEngage-related task — push notifications **and** in-app messaging (IAM). Its job is small and mechanical:

1. Detect which platform the customer's repo is.
2. Hand off to the matching platform spoke skill.
3. If multiple platforms are present (monorepo), ask the customer which one to work on this session.

**You don't answer PushEngage questions from here. You route.**

## Step 1 — Detect platform

Run these detection checks against the current working directory. Use a file-glob or shell `find` — do not read file contents yet.

| Platform | Detection markers (any one is sufficient) |
|---|---|
| **Flutter** (check FIRST) | `pubspec.yaml` containing a `flutter:` SDK block |
| **React Native** (check SECOND) | `package.json` containing a dependency on `react-native` or `expo` |
| **iOS** | `Podfile` · `Package.swift` · `**/*.xcodeproj` · `**/*.xcworkspace` |
| **Android** | `build.gradle` or `build.gradle.kts` at root AND `app/src/main/AndroidManifest.xml` |

**Precedence note:** Flutter and React Native projects also contain `ios/` and `android/` subfolders with their own Podfile / build.gradle. **Always check for Flutter and React Native FIRST.** A Flutter repo gets routed to `pushengage-flutter`, NOT to `pushengage-ios` or `pushengage-android`, even though those native markers exist. Same for React Native — route to `pushengage-react-native`.

Implementation hint:

```bash
# iOS
find . -maxdepth 4 \( -name "Podfile" -o -name "Package.swift" -o -name "*.xcodeproj" -o -name "*.xcworkspace" \) -print -quit
# Android
find . -maxdepth 4 \( -name "build.gradle" -o -name "build.gradle.kts" \) -print -quit
# Flutter
find . -maxdepth 4 -name "pubspec.yaml" -print -quit  # then grep -q "^  sdk: flutter" inside
# React Native
find . -maxdepth 4 -name "package.json" -print -quit  # then grep -q "react-native\|expo" inside
```

## Step 2 — Decide and hand off

| Detected | Action |
|---|---|
| Exactly **one** platform | Activate that platform's skill (below) and follow it. Use whatever mechanism your agent uses to load a skill by name; if your agent auto-activates skills by description, the matching `pushengage-<platform>` skill will already qualify — switch to it. Done. |
| **Multiple** platforms | Ask: "I see iOS and Android folders in this repo. Which platform are we working on this session?" Then activate that platform's skill. |
| **No** platforms detected | Ask: "I don't see iOS / Android / Flutter / React Native markers in this directory. Are you running this from the right folder?" Don't guess. |
| Platform whose skill isn't shipped yet | Tell the customer plainly: "This skill set currently covers iOS, Android, Flutter, and React Native. Your stack isn't supported." Don't fake it. |

Currently available platform skills:

- iOS → `pushengage-ios`
- Android → `pushengage-android`
- Flutter → `pushengage-flutter`
- React Native → `pushengage-react-native`

## Step 3 — Don't invent

Even after handing off, do not generate PushEngage API surface from training data. PushEngage is small enough that base-model recall is sparse or wrong. If the spoke doesn't cover what the customer is asking, say so and direct them to PushEngage support — don't fabricate.

## Cross-platform best practices

For best practices that apply regardless of platform (API key handling, permission timing, etc.), see `best-practices.md` in this skill folder. Spokes are encouraged to reference these rule IDs (`XP-BP-NN`) in their own audit findings.

## Dashboard-side tasks: the PushEngage MCP server

These skills cover **SDK integration and debugging in the customer's codebase**. Account-side operations — sending or scheduling notifications, creating segments, reading analytics, changing site settings — are handled by the official PushEngage MCP server (`@pushengage/mcp`), whose tools appear with names like `mcp__pushengage__*` when it is connected (the Claude Code plugin configures it automatically; other agents can add it from `https://github.com/awesomemotive/pushengage-mcp`).

- If the customer asks for a dashboard-side task and the MCP tools are available, use them.
- During debugging, the MCP tools can verify account-side state (site config, campaign payloads) that the code alone can't show — prefer that over asking the customer to screenshot the dashboard.
- If the MCP tools are not available, do NOT fabricate dashboard state. Tell the customer the steps to do it in the dashboard UI, or point them at the MCP server.
- First MCP use triggers a browser login to the customer's PushEngage account — that prompt is expected, not an error.

## In-App Messaging

In-app messages (dashboard-designed banners / modals / full-screen takeovers shown inside the app) are a separate channel from push and need only the App ID — no subscription, no notification permission. The platform-independent semantics (triggers, audience, action buttons, frequency caps, troubleshooting checklist) live in `in-app-messaging.md` in this folder; each spoke has its own `in-app-messaging.md` with the platform API and templates. Routing is the same as for push — detect the platform, hand off to the spoke, and the spoke reads its IAM file when the intent is "set up / debug in-app messages". IAM ships from iOS `1.1.0`, Android `1.0.0`, Flutter `1.1.0`, React Native `1.1.0`; older SDKs have no IAM API.

## Concepts

If you are not sure what a PushEngage term means (site, subscriber, profile ID, segment, App ID vs App Push Id, integration key, hash, in-app message, custom action, frequency cap), read `concepts.md` in this skill folder. Don't guess.
