# Android — Application Class Init Code

PushEngage init lives in the customer's custom `Application` subclass `onCreate()`. Builder pattern: `PushEngage.Builder().addContext(...).setAppId(...).build()`.

**Edit posture:** diff + confirm. Show the diff to the customer before applying.

---

## Discovery — does the customer already have an Application subclass?

Before touching files, detect by:

1. `grep -rn "extends Application" app/src/main/java app/src/main/kotlin 2>/dev/null` (Java).
2. `grep -rn ": Application()" app/src/main/java app/src/main/kotlin 2>/dev/null` (Kotlin).
3. Read `app/src/main/AndroidManifest.xml` and check for `android:name="..."` on the `<application>` tag.

| State | Action |
|---|---|
| No Application subclass | Create a new file — `PEApplication.kt` (or `.java` if the project is Java). Use the template below. |
| Has one (e.g. `class MyApplication : Application()`) | **Diff + confirm posture.** Add the PushEngage init lines inside the existing `onCreate()` after `super.onCreate()`. **Do NOT rename** the customer's class. |
| Manifest already declares `android:name="..."` matching the existing class | Leave manifest as-is. Init goes into the existing class. |
| Manifest has no `android:name` AND no Application subclass exists | Create `PEApplication`, then auto-edit + announce to add `android:name=".PEApplication"` to manifest. |

> ⚠️ ANDROID-AUDIT-10: if you create a new `Application` subclass and forget the manifest registration, `onCreate()` never runs and PushEngage never initializes.

---

## Kotlin template (preferred for new files)

File: `app/src/main/java/com/example/myapp/PEApplication.kt` (replace `com.example.myapp` with the customer's `applicationId`).

```kotlin
package com.example.myapp

import android.app.Application
import com.pushengage.pushengage.PushEngage

class PEApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        PushEngage.Builder()
            .addContext(applicationContext)
            .setAppId("APP_ID_FROM_DASHBOARD")
            .build()

        // ANDROID-BP-04: provide a real drawable. SDK silently falls back
        // to a bell icon if the drawable doesn't resolve.
        PushEngage.setSmallIconResource("ic_pushengage_notification")

        if (BuildConfig.DEBUG) {
            PushEngage.enableLogging(true)   // XP-BP-07 / ANDROID-BP-06
        }
    }
}
```

Replace `APP_ID_FROM_DASHBOARD` with the UUID copied in `dashboard-setup.md` Step 3.

## Java template

File: `app/src/main/java/com/example/myapp/PEApplication.java`.

```java
package com.example.myapp;

import android.app.Application;
import com.pushengage.pushengage.PushEngage;

public class PEApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        new PushEngage.Builder()
                .addContext(getApplicationContext())
                .setAppId("APP_ID_FROM_DASHBOARD")
                .build();

        // ANDROID-BP-04: provide a real drawable. SDK silently falls back
        // to a bell icon if the drawable doesn't resolve.
        PushEngage.setSmallIconResource("ic_pushengage_notification");

        if (BuildConfig.DEBUG) {
            PushEngage.enableLogging(true);  // XP-BP-07 / ANDROID-BP-06
        }
    }
}
```

---

## Manifest registration

Add `android:name=".PEApplication"` to the `<application>` element in `app/src/main/AndroidManifest.xml`. Auto-edit + announce posture:

```xml
<application
    android:name=".PEApplication"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    ... >
    <!-- ... activities ... -->
</application>
```

If the customer's existing Application class is named differently (e.g. `MyApplication`), the manifest already has `android:name=".MyApplication"`. Do not change it — init goes into that file instead.

> ⚠️ Without this manifest entry, Android instantiates a default `Application` instance and `PEApplication.onCreate()` never runs. ANDROID-AUDIT-10.

---

## Small-icon best practice (ANDROID-BP-04)

`setSmallIconResource(String)` takes a drawable resource **name**, NOT a resource ID. Make sure a drawable with that name exists in `res/drawable*/`; if it doesn't resolve, the SDK falls back to its bundled default bell icon (`ic_stat_notification_default`) instead (ANDROID-AUDIT-03).

What to do at this step:

1. Tell the customer the SDK needs a notification icon.
2. Ask: "Do you have a notification icon ready (transparent background, white-on-transparent, monochrome PNG or vector)?"
3. If yes — capture the drawable name and use it in `setSmallIconResource("<name>")`.
4. If no — leave the call referencing `ic_pushengage_notification` as a placeholder and **tell the customer they must add the drawable before shipping**. Provide format requirements:
   - Transparent background.
   - White-on-transparent (Android tints it per channel/system).
   - Monochrome (no colors render).
   - Format: vector `.xml` or `.png`.
   - Path: `res/drawable/ic_pushengage_notification.xml` (or `.png`).

**Do not silently invent a drawable** by writing a generated `.xml` file. The customer's branding decision is theirs.

---

## Existing-Application-class diff + confirm flow

If the customer already has, say, `MyApplication.kt`:

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // ... existing init: analytics SDKs, crash reporters, etc. ...
    }
}
```

Show the customer the proposed diff:

```diff
 class MyApplication : Application() {
     override fun onCreate() {
         super.onCreate()
         // ... existing init: analytics SDKs, crash reporters, etc. ...
+
+        PushEngage.Builder()
+            .addContext(applicationContext)
+            .setAppId("APP_ID_FROM_DASHBOARD")
+            .build()
+
+        PushEngage.setSmallIconResource("ic_pushengage_notification")
+        if (BuildConfig.DEBUG) {
+            PushEngage.enableLogging(true)
+        }
     }
 }
```

Ask: "Apply this diff?" Only edit on confirmation.

Insert PushEngage init **after** `super.onCreate()` and **after** any other SDK init that PushEngage might depend on indirectly (rare; typically PushEngage is independent). If the customer uses dependency injection (Hilt / Dagger), PushEngage init goes inside `onCreate()` regardless — it's not a `@Singleton` dependency.

---

## ANDROID-BP-03 reminder

Do NOT call `PushEngage.requestNotificationPermission(...)` here in `Application.onCreate()`. It requires a `ComponentActivity` (it won't even compile from `Application` context) — but the broader rule is to defer the permission request until the user sees value (a meaningful moment, mirroring XP-BP-03 from the iOS spoke). See `permission-flow.md`.

---

## Optional `0.1.0` hooks (mention as available — do NOT auto-insert)

Keep the init template above minimal. The following `0.1.0` additions are worth mentioning when relevant, but should not be added to `onCreate()` by default:

- **`PushEngage.setFcmConfigErrorListener { code, msg -> … }`** — the most useful new init-time debugging hook. Register it (ideally **before** `Builder.build()`) to surface FCM config mismatches between `google-services.json` and the dashboard service account early. Gate any logging it does behind `BuildConfig.DEBUG`, treat `msg` as potentially containing config identifiers, and marshal to the UI thread before touching UI. See ANDROID-BP-15. Example, only when the customer is debugging "no notifications at all":
  ```kotlin
  if (BuildConfig.DEBUG) {
      PushEngage.setFcmConfigErrorListener { code, msg ->
          Log.w("PushEngage", "FCM config error $code: $msg")
      }
  }
  ```
- **`PushEngage.setBadgeCount(int)`** — available for badge management (`0` cancels all notifications; `>0` sets a number on future SDK notifications). Not part of the init template.
- **User Identification API (`identify` / `logout` / `trackEvent`)** — post-init / post-subscribe surface. Document these in `best-practices.md` (ANDROID-BP-14) and concepts, NOT in the Application-class init template.

---

Next: `permission-flow.md` for the runtime-permission step. If the project uses Jetpack Compose, also read `jetpack-compose.md`.
