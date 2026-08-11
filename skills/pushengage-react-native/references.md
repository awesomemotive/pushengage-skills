# React Native Spoke — Allowed Documentation URLs

When this skill is active, you may fetch URLs in this list and ONLY URLs in this list. Other URLs — including PushEngage marketing pages, third-party blog posts, and Stack Overflow answers — are out of scope.

The reason: PushEngage's public surface area is small enough that hallucinating a URL is a real risk; an explicit allowlist closes that hole.

## Allowed URLs

| URL | Purpose | Caveats |
|---|---|---|
| `https://www.pushengage.com/documentation/setting-up-react-native-app-push-notification-using-pushengage.md` | Setup walkthrough | **Stale on four items:** import name (`'pushengage-react-native'` should be `'@pushengage/pushengage-react-native'`), NSE pod (`~>0.0.5` should be `pod 'PushEngageExtension', '1.0.0'`), NCE pod (`0.0.5` should be `pod 'PushEngageExtension', '1.0.0'`), NSE deployment target ("iOS 10+" should be 15.1+). Defer to this spoke's `version-matrix.md`. |
| `https://www.pushengage.com/api/mobile-sdk/react-native/sdk-reference.md` | API reference (RN-specific) | At capture time, depth varies. Verify against the published TypeScript source in the GitHub repo when in doubt. |
| `https://www.npmjs.com/package/@pushengage/pushengage-react-native` | Canonical published version + changelog | Source of truth for what version is actually shipped. |
| `https://github.com/awesomemotive/pushengage-react-native` | Source code + release tags | Source of truth for the TypeScript API surface. |
| `https://github.com/awesomemotive/pushengage-react-native/issues` | Bug tracker | For escalation when the known-issues table doesn't cover something. |

## Not allowed

- Marketing pages (e.g. `pushengage.com/best-react-native-push-notification-sdk/` or similar listicle URLs).
- Third-party blog posts, Stack Overflow answers, Medium articles — even if they look authoritative.
- Native iOS / Android documentation URLs from sibling spokes. The RN spoke borrows by file path (e.g., `../pushengage-android/firebase-setup.md`), not by URL.
- Any URL not in the table above. If a customer cites one, link them to the matching allowed URL and proceed from the spoke's content.

## When fetching

Always pass a specific extraction prompt — never a generic "summarize this page." The page content can drift over time; the spoke's content is the stable source. Treat the fetched page as supplementary; when it conflicts with `version-matrix.md`, the matrix wins.
