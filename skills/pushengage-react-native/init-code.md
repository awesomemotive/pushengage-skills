# Init Code (React Native)

Three pieces of init code: JS `index.js` / `index.tsx`, the iOS AppDelegate (`AppDelegate.swift` on RN ≥ 0.77 templates; `AppDelegate.mm` ObjC++ on apps upgraded from older RN), and (note) NO Android `MainApplication.kt` PushEngage init needed. Order matters — JS init triggers native init through the Turbo Module; the native delegate side handles delegate-level wiring.

## 1. JS — `index.js` / `index.tsx`

Canonical greenfield template (cross-platform):

```js
import { AppRegistry, Platform } from 'react-native';
import App from './App';
import { name as appName } from './app.json';
import PushEngage from '@pushengage/pushengage-react-native';

// Platform-conditional App ID — one PushEngage site per platform (KI-RN006).
const APP_ID = Platform.OS === 'ios'
  ? '<IOS_APP_ID_UUID>'
  : '<ANDROID_APP_ID_UUID>';

PushEngage.setAppId(APP_ID);

// Debug logging — wrap in __DEV__ (RN-BP-05).
if (__DEV__) {
  PushEngage.enableLogging(true);
}

AppRegistry.registerComponent(appName, () => App);
```

**Critical rules:**

1. **Import from `'@pushengage/pushengage-react-native'`**, NOT `'pushengage-react-native'` (without the scope) — an unscoped import won't resolve (RN-AUDIT-01 / KI-RN001).
2. Prefer calling `setAppId` at `index.js` module top-level, BEFORE `AppRegistry.registerComponent(...)`, so subscriber registration starts as early as possible (RN-BP-01 / RN-AUDIT-02). Calling it later — e.g. from `App.tsx` or a component effect, as the SDK's own example app does — is not an error and does not break subscription; early placement is simply the cleaner default.
3. Replace `<IOS_APP_ID_UUID>` and `<ANDROID_APP_ID_UUID>` with the actual UUIDs captured in `dashboard-setup.md`. Per XP-BP-01, recommend the customer later move them to a config file (`react-native-config`, `--env`, etc.), not hardcoded literals.
4. If shipping to one platform only, simplify:
   - iOS-only: `PushEngage.setAppId('<IOS_APP_ID_UUID>');`.
   - Android-only: `PushEngage.setAppId('<ANDROID_APP_ID_UUID>');`.
5. Wrap `enableLogging(true)` in `if (__DEV__) { ... }` (RN-BP-05) — always gate it so release builds don't emit logs.

### Deep-link listener — register in root component's `useEffect`

In your root component (the one passed to `AppRegistry.registerComponent`, typically `App.tsx`):

```tsx
import React from 'react';
import PushEngage from '@pushengage/pushengage-react-native';
import type { EventSubscription } from 'react-native';

export default function App() {
  React.useEffect(() => {
    // Runtime taps (app already running):
    const subscription: EventSubscription = PushEngage.onValueChanged((data) => {
      const { deepLink, data: payload } = data;
      // Customer-specific deep-link routing.
      // E.g., navigation.navigate(deepLink), or pattern-match on deepLink.
      console.log('Deep link:', deepLink, 'payload:', payload);
    });

    // Cold-boot tap (force-quit → tap → launch). iOS-only payload; Android resolves null.
    PushEngage.getInitialNotification().then((initial) => {
      if (initial) {
        console.log('Cold-boot deep link:', initial.deepLink, 'payload:', initial.data);
        // routeDeepLink(initial.deepLink, initial.data);
      }
    });

    return () => subscription.remove();   // RN-BP-08
  }, []);

  return /* ... your UI ... */;
}
```

**Cold-boot tap recovery (iOS) — new in 1.0.0.** `onValueChanged` alone does NOT deliver a tap that *launched* the app from a force-quit state on iOS: that delivery happens before JS can subscribe. Call `getInitialNotification()` once on startup to recover it. On Android it resolves `null` (cold-boot deep links arrive through the host activity intent filter), so the call is safe cross-platform. The cold-boot tap is routed **exclusively** to `getInitialNotification()` and is never replayed through `onValueChanged`, so the two channels never deliver the same payload — no dedup needed.

**Critical rules:**

1. Register inside the ROOT component's `useEffect`, with **empty dependency array `[]`**, BEFORE any auth gate, splash screen, or async startup work (RN-BP-02). Late registration → cold-boot tap loss (KI-RN003 — mirror of Flutter KI-F003).
2. ALWAYS return `() => subscription.remove()` from the effect (RN-BP-08). Hot-reloads and unmount cycles otherwise leak listeners → duplicate callbacks.
3. The handler receives `{ deepLink: string, data: { [k: string]: string } }`. `deepLink` is the URL string from the campaign; `data` is the custom payload.
4. Call `getInitialNotification()` once in the same startup effect alongside `onValueChanged` (RN-BP-13 / RN-AUDIT-17). On iOS this is the only way to recover a tap that launched the app from force-quit; on Android it resolves `null`.

## 2. iOS — AppDelegate (`AppDelegate.swift` or `AppDelegate.mm`)

**Check which delegate file exists before editing.** Since RN 0.77 the community template generates `AppDelegate.swift` by default — so greenfield apps on this spoke's RN ≥ 0.78 baseline have a Swift delegate and NO `.mm` file. Apps upgraded from older RN typically still carry `AppDelegate.mm` (ObjC++). Older guides show the ObjC++ (`.mm`) pattern; it's correct for `.mm` apps — just use the current pod pins.

### Swift variant (`AppDelegate.swift` — RN ≥ 0.77 templates)

Don't replace the RN-generated file — its scaffold differs across RN versions (0.77–0.78 subclass `RCTAppDelegate`; 0.79+ use the `RCTReactNativeFactory` pattern). Merge these PushEngage deltas into whatever scaffold the customer's RN version generated:

```swift
import PushEngage   // add to the imports

// 1. Enable swizzling in the delegate's initializer (add `override init()` if absent):
override init() {
    super.init()
    PushEngage.swizzleInjection(isEnabled: true)
}

// 2. Inside application(_:didFinishLaunchingWithOptions:), BEFORE the super /
//    factory call that boots React Native:
PushEngage.setInitialInfo(for: application, with: launchOptions)
PushEngage.setBadgeCount(count: 0)
```

Ordering rules are identical to the ObjC++ variant below: swizzling belongs in `init()`, and `setInitialInfo` must run before the call that starts React Native.

### Objective-C++ variant (`AppDelegate.mm` — upgraded apps)

Canonical template:

```objc
#import "AppDelegate.h"

#import <React/RCTBundleURLProvider.h>
#import <ReactAppDependencyProvider/RCTAppDependencyProvider.h>
@import PushEngage;

@implementation AppDelegate

- (instancetype)init {
  self = [super init];
  if (self) {
    [PushEngage swizzleInjectionWithIsEnabled:YES];
  }
  return self;
}

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  self.moduleName = @"<YourAppName>";
  self.dependencyProvider = [RCTAppDependencyProvider new];
  self.initialProps = @{};

  [PushEngage setInitialInfoFor:application with:launchOptions];
  [PushEngage setBadgeCountWithCount:0];

  return [super application:application
      didFinishLaunchingWithOptions:launchOptions];
}

- (void)application:(UIApplication *)application
    didReceiveRemoteNotification:(NSDictionary *)userInfo
          fetchCompletionHandler:
              (void (^)(UIBackgroundFetchResult))completionHandler {
  completionHandler(UIBackgroundFetchResultNewData);
}

// ... standard bundleURL / sourceURLForBridge boilerplate ...
@end
```

**Critical rules:**

1. `[PushEngage swizzleInjectionWithIsEnabled:YES]` MUST be inside `- (instancetype)init`, NOT inside `didFinishLaunchingWithOptions`. The SDK's swizzling registers BEFORE UIApplication queries the delegate. Wrong placement = double-init or silent no-op.
2. `[PushEngage setInitialInfoFor:application with:launchOptions]` MUST be called in `didFinishLaunchingWithOptions` BEFORE `super` is called. This captures the cold-boot launch options needed for replay.
3. Do NOT call `setAppId` from native code (ObjC or Swift). The App ID is set from JS (`index.js`). Calling from both = registration race.
4. `@import PushEngage;` works against the app-side `PushEngage` pod (pulled transitively by the RN plugin). As of 1.0.0 the two-pod split removed the need for the old `APPLICATION_EXTENSION_API_ONLY` / `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES` `post_install` overrides — the example app no longer sets them (see `install.md` Step 3b). If an older project still carries those overrides they are now harmless/obsolete; remove them when convenient (KI-RN008 context).
5. The `didReceiveRemoteNotification:` forward calls `completionHandler(UIBackgroundFetchResultNewData)` — this signals to iOS that the background-fetch slot was used productively. The SDK doesn't need a manual forward here in swizzling mode.
6. The native `setBadgeCount(count: 0)` / `[PushEngage setBadgeCountWithCount:0]` call shown above is the underlying iOS SDK API for clearing the badge at launch. It is unrelated to the new JS-facing `PushEngage.setBadgeCount(count)` bridge method (new in 1.0.0) for runtime badge updates from JS — keep this native launch call as-is.

## 3. Android — no `MainApplication.kt` PushEngage init needed

Unlike native Android (where the Android-spoke walks customers through creating an `Application` subclass to host PushEngage init), **the RN plugin handles native Android initialization through Turbo Module registration**. When JS calls `PushEngage.setAppId(...)`, the bridge forwards the App ID to the native Android plugin, which initializes the SDK on first call.

The customer does NOT need to:

- Modify `MainApplication.kt`'s `onCreate`.
- Call the PushEngage Android `Builder()` pattern manually.
- Add anything to `MainApplication.kt` beyond the standard RN setup.

**If the customer already has custom `MainApplication.kt` logic** (Hilt setup, crash reporting init, etc.), leave it alone. Don't add PushEngage init to it.

## Edit posture

- `index.js` / `index.tsx` — **diff + confirm.** Entry-point code; bad edits break the app.
- `ios/<YourApp>/AppDelegate.swift` / `AppDelegate.mm` — **diff + confirm.** Same reason.
- `App.tsx` (when adding the `onValueChanged` listener) — **diff + confirm.** Touching customer's root component.
- `ios/Podfile` — auto-edit + announce (covered in `install.md`).
- `android/app/build.gradle`, `android/AndroidManifest.xml` — auto-edit + announce (covered in `install.md`).

## Next

`permission-flow.md` for the runtime permission flow, then `nse-setup.md` for rich notifications.
