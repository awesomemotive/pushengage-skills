# Android — Jetpack Compose Specifics

This file covers Compose-only deltas. The main flow files (`init-code.md`, `permission-flow.md`) still apply — read those first.

---

## Compose detection (skill-internal)

Before surfacing this file to the customer, the Android spoke detects Compose:

```bash
grep -l "androidx.compose" app/build.gradle app/build.gradle.kts 2>/dev/null
```

If any match, Compose is in use → include `jetpack-compose.md` in the flow. If no match, skip this file.

The customer may also use Compose alongside an existing View-based codebase. Treat any detection of `androidx.compose` deps as "Compose is in scope."

---

## Composable vs `Application` name collision (ANDROID-BP-11)

Common pitfall: Compose's top-level composable is often named `MyApp()`:

```kotlin
@Composable
fun MyApp() {                       // ← composable
    // theme + nav host
}

setContent {
    MyApp()
}
```

If the customer also has an `Application` subclass named `MyApp`:

```kotlin
class MyApp : Application() {       // ← class with the same name
    override fun onCreate() { ... }
}
```

…the compiler will produce `ambiguous reference to MyApp` errors in the Compose code where the composable is referenced — Kotlin can't tell whether `MyApp` refers to the class or the function.

### Convention

- Composable: `AppContent()` (or `<AppName>Root()`).
- Application class: `MyApplication` or `PEApplication`.

If the customer already has `MyApp : Application`, **rename the composable**, not the class — the class name is referenced from the manifest (`android:name=".MyApp"`) and changing it touches more.

```kotlin
// Before
@Composable
fun MyApp() { ... }

// After
@Composable
fun AppContent() { ... }
```

ANDROID-AUDIT-12 catches this heuristically.

---

## Permission mechanism

The SDK's permission mechanism depends on the Activity's base class:

| Activity base | SDK mechanism |
|---|---|
| `FragmentActivity` subclass (incl. `AppCompatActivity`) | SDK adds a **headless fragment** to drive the system permission dialog. |
| Plain `androidx.activity.ComponentActivity` (common with pure Compose apps) | SDK launches a **transparent / translucent helper Activity** (`PEPermissionHelperActivity`, declared with `android:theme="@android:style/Theme.Translucent.NoTitleBar"`). |

In both cases the user sees only the system permission dialog — never an empty helper Activity. The customer doesn't need to declare anything; the SDK's manifest merges its own entries.

What this means for Compose customers:

- Pure-Compose apps with `MainActivity : ComponentActivity` get the **translucent-helper-Activity** path.
- Mixed apps with `MainActivity : AppCompatActivity` get the **headless-fragment** path.

Both work. The customer doesn't choose; the SDK picks based on the activity type passed to `requestNotificationPermission(this, ...)`.

---

## `NotificationPermissionGate` snapshot caveat (ANDROID-BP-12)

Tempting (and wrong) pattern:

```kotlin
@Composable
fun NotificationPermissionGate(content: @Composable () -> Unit) {
    val status = remember { PushEngage.getNotificationPermissionStatus() }   // ❌ snapshot
    if (status == "granted") {
        content()
    } else {
        SettingsDeepLinkCTA()
    }
}
```

`remember { }` captures a **snapshot at first composition**. If the user grants permission via the system Settings while the app is in the background, then returns to the app, this Composable does **not** recompose — the gate stays stuck on the CTA branch even though permission is now granted.

### Fix — drive from lifecycle

```kotlin
@Composable
fun NotificationPermissionGate(content: @Composable () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    var status by remember {
        mutableStateOf(PushEngage.getNotificationPermissionStatus())
    }

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) {
                status = PushEngage.getNotificationPermissionStatus()
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    if (status == "granted") {
        content()
    } else {
        SettingsDeepLinkCTA()
    }
}
```

The state is now refreshed on every `ON_RESUME`, which fires when the user returns from Settings.

---

## Calling `getSubscriberId` from Compose

Wrong pattern (a common mistake). It's wrong twice over — `coroutineScope` is undefined in that scope, AND `getSubscriberId` takes a `PushEngageResponseCallback`, which has two abstract methods (`onSuccess`, `onFailure`), so a Kotlin lambda does not SAM-convert and won't compile:

```kotlin
@Composable
fun SubscriberIdDisplay() {
    LaunchedEffect(Unit) {
        coroutineScope.launch {       // ❌ coroutineScope undefined in this scope
            PushEngage.getSubscriberId { id -> /* ... */ }   // ❌ lambda doesn't satisfy the 2-method callback interface
        }
    }
}
```

### Right pattern — bridge the callback with `suspendCancellableCoroutine`

```kotlin
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume

suspend fun fetchSubscriberId(): String? = suspendCancellableCoroutine { cont ->
    PushEngage.getSubscriberId(object : PushEngageResponseCallback {
        override fun onSuccess(responseObject: Any?) {
            cont.resume(responseObject as? String)
        }

        override fun onFailure(errorCode: Int?, errorMessage: String?) {
            cont.resume(null)
        }
    })
}

@Composable
fun SubscriberIdDisplay() {
    var subscriberId by remember { mutableStateOf<String?>(null) }
    LaunchedEffect(Unit) {
        subscriberId = fetchSubscriberId()
    }
    Text("Subscriber: ${subscriberId ?: "loading…"}")
}
```

The `LaunchedEffect` block is a coroutine scope tied to the composition's lifecycle. The `suspendCancellableCoroutine` turns the callback into a proper `suspend` function. No leak; cancels with the composition. The `object :` expression is required because `PushEngageResponseCallback` is a two-method interface (see `version-matrix.md`).

---

## Other Compose call-site notes

- `requestNotificationPermission(this, ...)` from a Compose activity works the same way as in `permission-flow.md` — `this` refers to the enclosing `ComponentActivity`. You can also pass `LocalContext.current as ComponentActivity` from inside a composable, but prefer hoisting the call up to the activity to keep composables side-effect-free.
- Avoid calling `requestNotificationPermission` inside `LaunchedEffect(Unit)` at the root of `setContent { }`. That's the Compose equivalent of "calling from `onCreate`" — fires on every cold start, before the user sees value. ANDROID-BP-03.

---

Next: continue back to the orchestrator. Permission flow main file is `permission-flow.md`; this file is the Compose delta.
