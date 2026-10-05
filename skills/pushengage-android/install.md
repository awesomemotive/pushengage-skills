# Android SDK Install

Gradle setup. The Android SDK is published on **JitPack** (not Maven Central): coordinate `com.github.awesomemotive:pushengage-android-sdk:1.0.0`. Authoritative against `version-matrix.md`.

## Decision rule — Groovy DSL vs Kotlin DSL

Detect which Gradle DSL the project uses; **do not migrate between them**.

| Detected | Use |
|---|---|
| `build.gradle` (Groovy) at root, `app/build.gradle` | Groovy DSL path below |
| `build.gradle.kts` at root, `app/build.gradle.kts` | Kotlin DSL path below |
| Mixed (rare) | Match each file's existing extension |

Detect which repository configuration the project uses:

| Detected | Repo block lives in |
|---|---|
| `settings.gradle` / `settings.gradle.kts` contains a `dependencyResolutionManagement { repositories { ... } }` block | **`settings.gradle*`** (centralized — modern Android Studio default) |
| No such block; project-level `build.gradle*` has `allprojects { repositories { ... } }` | **Project-level `build.gradle*`** |

JitPack lives wherever the customer's other repositories live — do not duplicate.

## Mandatory edits (auto-edit + announce posture)

1. Add **JitPack repo** — to `settings.gradle*` or project-level `build.gradle*`.
2. Add **google-services classpath** — to project-level `build.gradle*`.
3. Add **google-services plugin** — to `app/build.gradle*`, **AFTER** `com.android.application` (ANDROID-BP-08).
4. Add **firebase-bom + firebase-messaging** dependencies — to `app/build.gradle*`.
5. Add **PushEngage SDK** dependency — to `app/build.gradle*`.
6. Add **`POST_NOTIFICATIONS`** permission — to `app/src/main/AndroidManifest.xml` (ANDROID-BP-02 / KI-A001).

---

## Groovy DSL path

### Project-level `build.gradle`

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.gms:google-services:4.4.2'
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

(If the project uses centralized repos via `settings.gradle` — see next section — omit the `allprojects { repositories { ... } }` block here.)

### Centralized-repo variant — `settings.gradle`

If `settings.gradle` already has `dependencyResolutionManagement`, add JitPack there instead of `allprojects`:

```groovy
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### App-level `app/build.gradle`

```groovy
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'   // only if the project uses Kotlin
    id 'com.google.gms.google-services' // MUST be AFTER com.android.application
}

android {
    namespace 'com.example.myapp'
    compileSdk 34

    defaultConfig {
        applicationId 'com.example.myapp'
        minSdkVersion 21          // SDK supports 16; recommend 21+
        targetSdkVersion 34
    }
}

dependencies {
    implementation platform('com.google.firebase:firebase-bom:33.5.1')
    implementation 'com.google.firebase:firebase-messaging'
    implementation 'com.github.awesomemotive:pushengage-android-sdk:1.0.0'
}
```

---

## Kotlin DSL path

### Project-level `build.gradle.kts`

```kotlin
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath("com.google.gms:google-services:4.4.2")
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

(If the project uses centralized repos via `settings.gradle.kts` — see next section — omit the `allprojects { repositories { ... } }` block here.)

### Centralized-repo variant — `settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

### App-level `app/build.gradle.kts`

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.gms.google-services")  // MUST be AFTER com.android.application
}

android {
    namespace = "com.example.myapp"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 21
        targetSdk = 34
    }
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.5.1"))
    implementation("com.google.firebase:firebase-messaging")
    implementation("com.github.awesomemotive:pushengage-android-sdk:1.0.0")
}
```

---

## AndroidManifest.xml — `POST_NOTIFICATIONS`

Declare the Android 13+ (API 33+) runtime notification permission explicitly. The SDK's own library manifest already declares it and manifest merger propagates it into the app, so this entry is belt-and-braces rather than load-bearing — but declare it anyway: it documents intent and protects against `tools:node="remove"` / merger edge cases (KI-A001 / ANDROID-AUDIT-01 / ANDROID-BP-02).

Edit `app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Android 13+ runtime permission -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:name=".PEApplication"
        android:icon="@mipmap/ic_launcher"
        ... >
        <!-- ... activities ... -->
    </application>

</manifest>
```

The `android:name=".PEApplication"` registration is added in `init-code.md` Step 3. If the customer already has an Application subclass, the existing `android:name=...` value is kept (do not overwrite).

## Version pin

Always pin **exactly** to the version in `version-matrix.md`:
- PushEngage SDK: `1.0.0`
- google-services plugin: `4.4.2`
- firebase-bom: `33.5.1`

Do not use range operators (violates XP-BP-06 / ANDROID-AUDIT-08).

> ℹ️ The SDK itself builds against older Firebase versions (`google-services:4.3.15`, `firebase-bom:26.1.1`; its README's install snippet shows `4.3.10`); those work, but for modern apps targeting SDK 34 prefer the newer pins from `version-matrix.md` (above). Firebase BOMs are backwards compatible.

## Sanity check

After applying the edits, instruct the customer to run:

```bash
./gradlew :app:assembleDebug
```

Expected: succeeds. Common failures and their causes:

| Error | Cause | Fix |
|---|---|---|
| `File google-services.json is missing. The Google Services Plugin cannot function without it.` | `google-services.json` is at project root, not `app/`. | KI-A006 / ANDROID-AUDIT-04. Move file to `app/google-services.json`. |
| `Plugin [id: 'com.google.gms.google-services'] was not found` | Classpath not added to project-level `build.gradle*`. | Re-check Step 2 — add the classpath. |
| `google-services plugin not applied` / NPE during `:app:processDebugGoogleServices` | `id('com.google.gms.google-services')` placed BEFORE `id('com.android.application')` in plugins block. | KI-A007 / ANDROID-AUDIT-05 / ANDROID-BP-08. Reorder so google-services comes AFTER `com.android.application`. |
| `Failed to resolve: com.github.awesomemotive:pushengage-android-sdk` | JitPack repo not added. | Re-check Step 1 — add JitPack to `settings.gradle*` or `allprojects`. |

---

Next: write the init code (`init-code.md`).
