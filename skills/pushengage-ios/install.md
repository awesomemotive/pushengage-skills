# iOS SDK Install

Two package managers supported: Swift Package Manager (SPM) and CocoaPods. Both authoritative against `version-matrix.md`.

## Decision rule

| Customer's repo state | Use |
|---|---|
| Already has `Podfile` / `Pods/` | CocoaPods |
| Already has `Package.swift` / `*.xcodeproj` references SPM packages, no Podfile | SPM |
| Both present | CocoaPods (they're already invested) |
| Neither (greenfield) | **SPM** (preferred — simpler, no per-pod hooks) |

## SPM path

1. In Xcode → **File → Add Package Dependencies…**
2. Paste: `https://github.com/awesomemotive/pushengage-ios-sdk`
3. Click **Add Package**.
4. Under **Add to Target**, link the **`PushEngage`** product to the main app target. If you have NSE/NCE targets, link the **`PushEngageExtension`** product to those (not `PushEngage`). (If NSE/NCE targets exist already, set them up now — but most flows add extensions *after* this step, so come back.)
5. Click **Add Package**.

Auto-edit posture: instruct user via Xcode UI. SPM dependencies live in `xcshareddata/swiftpm/Package.resolved` — Xcode manages this; do not hand-edit.

## CocoaPods path

### Bootstrap (if no Podfile exists yet)

```bash
sudo gem install cocoapods   # if not installed
cd <project-root>
pod init
```

### Edit Podfile

Auto-edit the Podfile to add the PushEngage dependency for the app target:

```ruby
target 'YourMainTarget' do
  use_frameworks!
  pod 'PushEngage', '1.1.0'
end
```

Replace `YourMainTarget` with the customer's actual main target name (read it from the existing Podfile or `.xcodeproj`).

> **Post-install hook (IOS-BP-12) — no longer required for `1.0.0`.** Earlier baselines mandated an `APPLICATION_EXTENSION_API_ONLY = 'No'` post-install hook because the single shared pod imported non-extension API into extension targets. With the `1.0.0` two-module split, your extension targets link only the extension-safe `PushEngageExtension` pod, so this hook is usually unnecessary. Keep it only as a fallback if your `PushEngage` app pod is (incorrectly) linked into an extension target, or if you still see `APPLICATION_EXTENSION_API_ONLY` errors:
>
> ```ruby
> post_install do |installer|
>   installer.pods_project.targets.each do |target|
>     target.build_configurations.each do |config|
>       config.build_settings['APPLICATION_EXTENSION_API_ONLY'] = 'No'
>     end
>   end
> end
> ```

### Install

```bash
cd <project-root>
pod repo update
pod install
```

Then open the generated `.xcworkspace` (not `.xcodeproj`) in Xcode.

## Version pin

Always pin **exactly** to the version in `version-matrix.md` (currently `1.1.0`; requires deployment target iOS 15.0+). Do not use `~>` or other range operators (violates XP-BP-06). This applies to both pods: `pod 'PushEngage', '1.1.0'` on the app target and `pod 'PushEngageExtension', '1.1.0'` on any NSE/NCE target.

## Sanity check

After install (either path), instruct the customer to run a clean build:

```bash
xcodebuild -workspace YourApp.xcworkspace -scheme YourApp -configuration Debug build | tail -40
```

Expected: succeeds.

If it fails with `User Script Sandboxing` errors (Xcode 15+), apply IOS-BP-13:

- **Build Settings → User Script Sandboxing → No** (per-target, for the main app and any extensions).

---

Next: write the init code (`init-code.md`).
