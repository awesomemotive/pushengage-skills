# Flutter Spoke — Allowed Documentation URLs

When this skill is active, you may fetch URLs in this list and ONLY URLs in this list. Other URLs — including PushEngage marketing pages, third-party blog posts, and Stack Overflow answers — are out of scope.

The reason: PushEngage's public surface area is small enough that hallucinating a URL is a real risk; an explicit allowlist closes that hole.

## Allowed URLs

| URL | Purpose | Caveats |
|---|---|---|
| `https://www.pushengage.com/documentation/setting-up-app-push-notification-in-flutter-using-pushengage.md` | Full Flutter setup walkthrough | **Stale on three version pins:** `pushengage_flutter_sdk ^0.0.1` (actual `^1.0.0`), NSE pod `~>0.0.4` (actual `pod 'PushEngageExtension', '1.0.0'`), NCE pod `0.0.4` (actual `pod 'PushEngageExtension', '1.0.0'`). Defer to this spoke's `version-matrix.md`. |
| `https://www.pushengage.com/api/mobile-sdk/flutter/sdk-reference.md` | API quickstart landing | At capture time, returned a generic landing card with no method-level detail. May be populated later. Verify before extraction. |
| `https://pub.dev/packages/pushengage_flutter_sdk` | Canonical published version + changelog | Source of truth for what version is actually shipped. |
| `https://github.com/awesomemotive/pushengage-flutter-sdk` | Source code + release tags | Source of truth for the public Dart API surface. The published Dart API is the canonical method list. |
| `https://github.com/awesomemotive/pushengage-flutter-sdk/releases` | Release notes | Linked from the repo changelog. |

## Not allowed

- Marketing pages (e.g. `pushengage.com/best-flutter-push-notification-sdk/` or similar listicle URLs).
- Third-party blog posts, Stack Overflow answers, Medium articles — even if they look authoritative.
- The native iOS and Android documentation URLs from the sibling spokes. The Flutter spoke borrows content from those spokes by file path (e.g., `../pushengage-android/firebase-setup.md`), not by URL.
- Any URL not in the table above. If a customer cites one, link them to the matching allowed URL and proceed from the spoke's content.

## When fetching

Always pass a specific extraction prompt — never a generic "summarize this page." The page content can drift over time; the spoke's content (this file's neighbors) is the stable source. Treat the fetched page as supplementary; when it conflicts with `version-matrix.md`, the matrix wins.
