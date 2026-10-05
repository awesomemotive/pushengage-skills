# In-App Messaging (IAM) — cross-platform reference

Read this when a customer asks about in-app messages (banners, modals, full-screen takeovers rendered inside the running app). It covers the behavior that is **identical on every platform**. Platform-specific API, setup and code templates live in each spoke's `in-app-messaging.md`:

- iOS → `../pushengage-ios/in-app-messaging.md`
- Android → `../pushengage-android/in-app-messaging.md`
- Flutter → `../pushengage-flutter/in-app-messaging.md`
- React Native → `../pushengage-react-native/in-app-messaging.md`

Customer-facing guide: `https://www.pushengage.com/api/mobile-sdk/in-app-messaging.md` (allowlisted in every spoke's `references.md`) — **not yet published as of 2026-10-05 (404)**. Until it is live, the released SDK READMEs' In-App Messaging sections on GitHub are the public source of truth; this file and the spoke files were validated against the released SDKs (iOS `1.1.0`, Android `1.0.0`, Flutter `1.1.0`, React Native `1.1.0`). Do not invent semantics beyond these.

## What it is

Dashboard-designed HTML messages that the SDK displays inside the app at a moment the campaign defines — onboarding prompts, promotions, feature announcements, or a pre-permission explainer before the system notification prompt. It is a **separate channel from push**.

## Availability (first SDK version with IAM)

| SDK | Minimum version | Notes |
|---|---|---|
| iOS | `1.1.0` | Min deployment target rises to **iOS 15.0** (from 12.0). |
| Android | `1.0.0` | No setup change. |
| Flutter | `1.1.0` | Transitive iOS `1.1.0` / Android `1.0.0`; Podfile needs `platform :ios, '15.0'`. |
| React Native | `1.1.0` | Transitive iOS `1.1.0` / Android `1.0.0`. |

Older versions have **no IAM API at all** — a customer on iOS `1.0.0` or Android `0.1.0` asking about `triggerIAMEvent` must upgrade first (check the spoke's `version-matrix.md`).

## Works without a push subscription

IAM needs only the **App ID** (`setAppID` / `setAppId`). Campaigns sync and display even if the user never granted notification permission and never subscribed. Do NOT tell a customer to call `subscribe()` or `requestNotificationPermission()` as an IAM prerequisite.

**The one exception:** audience rules over *subscriber data* (segments, `attr.*`, city/state/geo_country, `has_unsubscribed`) are **held back, not evaluated**, until the SDK has fetched the subscriber record at least once — which requires push set up and the user subscribed. In an IAM-only integration those campaigns never become eligible; on a fresh install they typically first evaluate on the **second** app open. Device-data rules are unaffected. (XP-BP-10.)

## How it works

1. The customer creates and publishes a campaign on the dashboard: content (HTML), position, trigger, audience, frequency, date window, priority, action buttons.
2. The SDK syncs campaigns on SDK init and then roughly every 6 hours (iOS can additionally opt in to background refresh). Sync is gated on the site having IAM enabled; if IAM is disabled for the site, the SDK purges local campaigns and shows nothing.
3. When the trigger fires and the campaign is eligible (date window, frequency cap, audience), the SDK shows it — one message at a time — and reports impressions and button clicks.

### Dashboard prerequisites

The SDK docs do **not** document the dashboard click-path for creating IAM campaigns. Do not invent menu names. Tell the customer what a campaign needs and let them locate the In-App Messaging area of the dashboard (or direct them to PushEngage support):

- IAM enabled for the site.
- At least one **published, active** campaign.
- Every campaign has a **trigger** (app open, or a custom event name). A campaign with no trigger can never show.
- Every campaign has its own way out — a button wired to a **Dismiss** action, or an auto-dismiss duration. The SDK draws **no** close button (XP-BP-11).
- Each `custom` button has an **Action name** on the dashboard; the app's custom-action handler switches on that name (delivered as `parameters["action"]`), so the two must match exactly.

If the PushEngage MCP tools are connected, check whether any of them expose IAM campaigns before asking the customer to screenshot the dashboard; at the time of writing they do not, so don't fabricate campaign state.

## Triggers

Exactly two kinds:

**App open** — fired by the SDK itself, **once per process lifetime** (at SDK init or the first screen appearing, whichever comes first). Backgrounding and returning does **not** re-fire it; only a fresh launch does. To re-test an app-open campaign, force-quit and relaunch. An optional delay is measured from the app-open instant (a slow sync eats into it, backgrounding pauses it).

**Custom event** — the app reports an event and campaigns with a matching trigger name are evaluated:

| Platform | Call |
|---|---|
| iOS | `PushEngage.triggerIAMEvent(eventName:parameters:completionHandler:)` |
| Android | `PushEngage.triggerIAMEvent(eventName[, parameters][, callback])` |
| Flutter | `PushEngage.triggerIAMEvent(eventName, [parameters])` |
| React Native | `PushEngage.triggerIAMEvent(eventName, parameters?)` |

Rules that hold everywhere (XP-BP-08):

- The event name must equal the dashboard trigger event **exactly, including case**.
- A blank name is rejected with code `400` / `Event name is required` — never silently ignored.
- **Success means the event was accepted, not that a message was shown.** Matching and display are asynchronous.
- Parameter values are **stringified before matching**. The condition's *type* on the dashboard (`string`, `number`, `version`, `boolean`) decides how the text is compared: give numeric comparisons a `number` condition (then `120`, `"120"` and `120.0` are equal and `greater than` orders numerically). A `string` condition compares text as-is; `contains` is text-only.
- A campaign with no trigger conditions matches every occurrence of its event. Extra parameters the campaign doesn't name are ignored. A condition naming a parameter the app didn't send is "missing": positive operators (`equals`, `greater than`, `contains`, `in`) fail; negative ones (`not equals`, `not in`) pass.
- All conditions must pass unless the trigger's match mode is *any*.
- **Malformed campaigns fail closed** (never shown). The SDK logs the reason with the campaign id.

There is no built-in screen-view or session trigger. A "screen view" is just a custom event the app chooses to fire.

## Audience targeting

Groups are OR'd; conditions inside a group are AND'd (either level switchable on the dashboard). No audience = everyone.

Device fields (always resolvable, including on a fresh install):

| Field | Type | Notes |
|---|---|---|
| `platform` | string | Exactly `iOS` or `Android` — case-sensitive. `android` never matches. |
| `os_version` | version | Marketing version (`17.4`, `14`), not the Android API level. |
| `app_version` | version | The app's own version string. |
| `language` | string | Bare code (`en`), never `en-US`. |
| `device_region` | string | Locale region code (`US`). Distinct from IP-derived `geo_country`. |
| `timezone` | string | IANA identifier (`Asia/Kolkata`). |
| `notification_enabled` | boolean | Whether notification permission is currently granted. |

Subscriber fields (deferred until the subscriber record exists — see above): `segments` (uses *includes* / *excludes* only), `city`, `state`, `geo_country`, `has_unsubscribed`, `attr.<key>` (the `attr.` prefix is required).

Operators: `equals`, `not equals`, `greater than`, `less than`, `greater or equal`, `less or equal`, `in`, `not in`, `contains`, plus *includes* / *excludes* for `segments`. Version fields compare component-wise (`14` beats `8.1.0`). String comparison is exact and case-sensitive. Unknown/malformed audiences fail closed.

## Action buttons

Four wire types, lowercase, matched exactly:

| Type | Who handles | What happens |
|---|---|---|
| `open_url` | SDK | Opens in the **external browser**. Missing scheme → `https` assumed. Anything not `http`/`https` (including custom app schemes like `myapp://`) is **refused**. |
| `request_notification_permission` | SDK | Shows the system prompt; on grant the SDK registers the device as a push subscriber. No host code (Android: do NOT forward `onRequestPermissionsResult`). |
| `dismiss` | SDK | Closes the message. |
| `custom` | **Host app** | Delivers `actionId` + string-valued `parameters` to the app's handler/stream, then closes the message. **Route on `parameters["action"]`** — the button's Action name from the dashboard. `actionId` is a dashboard-generated internal key (e.g. `t04B05`); never route on it. |

Rules:

- **Any recognised button dismisses the message**, including `custom` and including a button whose action failed (bad URL). Every tap is reported as a click, `dismiss` included.
- An **unrecognised action type** (typo, or newer than the installed SDK) is inert: nothing happens, no click, message stays up. Only that button is affected.
- A `custom` button with **no handler registered** still counts a click and still closes — the app just never hears about it.
- `parameters` are forwarded verbatim (plus the `action` entry carrying the Action name); a `url` parameter on a `custom` action is **not** opened. In-app navigation / deep links go through `custom` actions handled by the app (XP-BP-09). IAM never flows through the push deep-link listeners (`setNotificationOpenHandler`, `deepLinkStream`, `onValueChanged`).

Custom-action receivers per platform:

| Platform | Receive custom actions |
|---|---|
| iOS | `PushEngage.setIAMCustomActionHandler { actionId, parameters in … }` (single slot) |
| Android | `PushEngage.setIAMCustomActionHandler(IAMCustomActionHandler)` (`null` clears) |
| Flutter | `PushEngage.onIAMCustomAction.listen(...)` → `StreamSubscription` |
| React Native | `PushEngage.onIAMCustomAction(cb)` → `EventSubscription` |

## Display behavior

- **Positions:** `top` (banner), `bottom` (banner), `center` (modal), `full` (full screen). Banners and full-screen messages must carry their own close control in the HTML.
- **Date window:** inclusive, absolute UTC, re-checked at display time (not sync time).
- **Auto-dismiss** (`displayDuration` seconds; `0` = stays until acted on). Countdown starts at presentation, not when the HTML finishes rendering. Rotation doesn't restart it.
- **Tap outside to dismiss** (`shouldDismissOnTap`, off by default) — same at all positions; taps on the message itself always go to the content; an outside tap never reaches the app behind.
- **Swipe to dismiss** is Android-only, modal (`center`) only. iOS has no swipe dismissal.
- **Backgrounding:** on Android the message closes and the display still counts against the frequency cap; behavior on iOS differs today — don't promise either way.
- **Priority & queue:** lower number shows first; the next message appears only after the current one is dismissed; a newly eligible high-priority campaign never interrupts one on screen. Give campaigns **distinct** priorities (tie-breaking differs by platform). iOS queues at most 10.
- **Analytics:** only impressions and button clicks are sent (dismiss buttons included — the dashboard's **Closes** metric is derived from clicks whose type is `dismiss`). Auto-dismiss, tap-outside, swipe and background dismissals are **not** reported.

## Frequency caps

| Cap | Behavior |
|---|---|
| No cap | Shows every time it is eligible. |
| One time | Once per device, ever. To re-test: clear app data / reinstall. |
| Capped | Up to N total **and** at most once per interval (no interval → pure count cap). |
| Recurring | Unlimited, spaced by at least the interval. No total cap — use Capped for one. |

Counts are stored per device. A cap that can't be applied (Capped without a count, Recurring without an interval) blocks the message rather than showing it uncapped.

## Troubleshooting checklist (all platforms)

Work down in order; each rejection is logged with the campaign id when SDK logging is on:

1. **SDK version has IAM?** See the availability table. Wrapper apps: check the *transitive* native versions too.
2. **Has the campaign synced?** Sync happens on init and ~every 6 h. Force-quit and relaunch after creating a campaign.
3. **Is IAM enabled for the site and the campaign published/active?** Disabled site → local campaigns purged.
4. **Does the trigger event name match character-for-character?** App-open campaigns: they fire once per process — relaunch, don't just foreground.
5. **Inside the date window?**
6. **Frequency cap reached?** A One-time campaign that already showed never shows again on that device.
7. **Audience rules match?** Device rules always evaluate; subscriber-data rules are skipped until the subscriber record has been fetched (needs a push subscription).
8. **Another message already on screen?** Only one at a time.
9. **Button does nothing?** Check the action type spelling (`open_url`, `request_notification_permission`, `dismiss`, `custom`) and, for `open_url`, that the URL is http(s).
10. **Custom action never reaches the app?** Handler/listener registered (and, on iOS, not overwritten by a later `setIAMCustomActionHandler` call)? Handler switching on `actionId` instead of `parameters["action"]`? (`actionId` is an internal key like `t04B05`, not the Action name.)

With logging enabled the SDK also dumps a snapshot of the on-device campaign store (counts, per-campaign trigger / position / display count / actions / HTML) — Android on IAM init, iOS once per session on app open. See `../pushengage-debug/log-readers.md` → In-App Messaging and `../pushengage-debug/diagnostic-tree.md` → Branch 8.
