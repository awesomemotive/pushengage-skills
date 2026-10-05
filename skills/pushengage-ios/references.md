# iOS Spoke — Allowed Documentation URLs

When this skill is active, you may fetch URLs in this list and ONLY URLs in this list. Other URLs — including PushEngage marketing pages, third-party blog posts, and Stack Overflow answers — are out of scope.

The reason: PushEngage's public surface area is small enough that hallucinating a URL is a real risk; an explicit allowlist closes that hole.

## Allowed URLs

| URL | Purpose | Caveats |
|---|---|---|
| `https://www.pushengage.com/documentation/how-to-set-up-ios-app-push-notifications-with-pushengage.md` | Full iOS integration walkthrough | For the current API spellings (e.g. `enableLogging`) and the `.p12` dashboard flow, defer to this spoke's content. |
| `https://www.pushengage.com/documentation/guide-to-creating-ios-apns-certificate.md` | APNs cert generation in Apple Developer portal + Keychain export to `.p12` | Authoritative for cert generation steps; thin on troubleshooting. |
| `https://www.pushengage.com/api/mobile-sdk/ios/sdk-reference.md` | API reference (iOS-specific) | At capture time, depth varies. Verify against the Swift source in the GitHub repo when in doubt. |
| `https://github.com/awesomemotive/pushengage-ios-sdk` | SPM package URL + source of truth for the public Swift API | Pinned tag is the version in `version-matrix.md` (currently `1.1.0`). Two products live at this tag: `PushEngage` (app target) and `PushEngageExtension` (NSE/NCE targets). |
| `https://www.pushengage.com/api/mobile-sdk/in-app-messaging.md` | Cross-platform In-App Messaging guide (triggers, audience, actions, frequency caps, troubleshooting) | **Not yet published as of 2026-10-05 (returns 404)** — the page exists only on the api-docs `task/iam-app` branch. Until it is live, treat this spoke's `in-app-messaging.md`, `../pushengage/in-app-messaging.md`, and the released SDK README's In-App Messaging section (GitHub URL above) as the source of truth. Does NOT document the dashboard click-path for creating campaigns. |

## Not allowed

- Marketing pages (e.g. `pushengage.com/how-to-use-an-ios-push-notifications-sdk/`).
- Any URL not in the table above. If a customer cites one, link them to the matching allowed URL and proceed from the spoke's content.

## Rule

When fetching, always pass a specific extraction prompt — never generic "summarize this page." The page content can drift; the spoke's content (this file's neighbors) is the stable source.
