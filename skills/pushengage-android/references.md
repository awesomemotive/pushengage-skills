# Android Spoke — Allowed Documentation URLs

When this skill is active, you may fetch URLs in this list and ONLY URLs in this list. Other URLs — including PushEngage marketing pages, third-party blog posts, and Stack Overflow answers — are out of scope.

The reason: PushEngage's public surface area is small enough that hallucinating a URL is a real risk; an explicit allowlist closes that hole.

## Allowed URLs

| URL | Purpose | Caveats |
|---|---|---|
| `https://www.pushengage.com/documentation/android-push-notification-sdk-setup.md` | Full Android setup walkthrough | For current Firebase pins and the `POST_NOTIFICATIONS` manifest entry, defer to this spoke's `version-matrix.md` + `install.md`. |
| `https://www.pushengage.com/api/mobile-sdk/android/sdk-reference.md` | Detailed Android API reference | URL as documented in the SDK repo's own README ("Full API reference"). If the page 404s, fall back to `version-matrix.md`. |
| `https://www.pushengage.com/api/mobile-sdk/in-app-messaging.md` | Cross-platform In-App Messaging guide (triggers, audience, actions, frequency caps, troubleshooting) | **Not yet published as of 2026-10-05 (returns 404)** — the page exists only on the api-docs `task/iam-app` branch. Until it is live, treat this spoke's `in-app-messaging.md`, `../pushengage/in-app-messaging.md`, and the released SDK README's In-App Messaging section (GitHub URL above) as the source of truth. Does NOT document the dashboard click-path for creating campaigns. |
| `https://github.com/awesomemotive/pushengage-android-sdk` | Source of truth for the JitPack tag + public Kotlin/Java API | Pinned tag is the version in `version-matrix.md` (currently `1.0.0`). SPM-equivalent role for Android (the JitPack coordinate resolves through this repo). |

## Not allowed

- Marketing pages (e.g. `pushengage.com/best-android-push-notification-sdk/` or similar listicle URLs).
- Third-party blog posts, Stack Overflow answers, Medium articles — even if they look authoritative.
- Any URL not in the table above. If a customer cites one, link them to the matching allowed URL and proceed from the spoke's content.

## When fetching

Always pass a specific extraction prompt — never a generic "summarize this page." The page content can drift over time; the spoke's content (this file's neighbors) is the stable source. Treat the fetched page as supplementary; when it conflicts with `version-matrix.md`, the matrix wins.
