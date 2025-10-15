## Side effects

A `side-effect` is a change to the state of the app that happens outside the scope of a composable function. Due to composables' lifecycle and properties such as unpredictable recompositions, executing recompositions of composables in different orders, or recompositions that can be discarded, composables should ideally be side-effect free.

However, sometimes side-effects are necessary, for example, to trigger a one-off event such as showing a snackbar or navigate to another screen given a certain state condition. These actions should be called from a controlled environment that is aware of the lifecycle of the composable. In this page, you'll learn about the different side-effect APIs Jetpack Compose offers.

> **Key Term:** *An effect is a composable function that doesn't emit UI and causes side effects to run when a composition completes.*

## 🧩 What is `LaunchedEffect`?

`LaunchedEffect` is a **side-effect API** in Jetpack Compose used to **launch coroutines** tied to a composable’s **lifecycle**.

It allows you to safely run **suspend functions** (like data loading, animations, or network calls) when your composable **enters the composition** or when certain **keys change**.

---

## ⚙️ Basic Syntax

```kotlin
LaunchedEffect(key1, key2, ...) {
    // Suspend or side-effect logic here
}
```

* **Keys**: One or more values that control when the effect runs or restarts.
* **Block**: A suspend lambda executed inside a coroutine.

---

## 🧠 Key Concepts

| Concept                 | Explanation                                                    |
| ----------------------- | -------------------------------------------------------------- |
| **Purpose**             | Run suspend code that depends on composition or state changes. |
| **Coroutine scope**     | Automatically created and managed by Compose.                  |
| **Lifecycle awareness** | Cancelled when the composable leaves composition.              |
| **Re-run condition**    | When any of the keys change, the effect restarts.              |
| **Suspend support**     | Can safely call suspend functions (e.g., from ViewModel).      |

---

## 🪄 Lifecycle Behavior

| Event                         | Effect                                    |
| ----------------------------- | ----------------------------------------- |
| Composable enters composition | Starts coroutine and runs block.          |
| Key changes                   | Cancels old coroutine and starts new one. |
| Composable leaves composition | Cancels coroutine automatically.          |

---

## ⚡ Usage Patterns

### 1️⃣ Run Once (on first composition)

Use `Unit` as the key if you only want the effect to run once.

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadData() // Runs once
}
```

### 2️⃣ Run When State Changes

Pass a variable as the key to re-run when that variable changes.

```kotlin
LaunchedEffect(userId) {
    viewModel.loadUserData(userId) // Re-runs when userId changes
}
```

### 3️⃣ Perform Delayed Actions or Animations

```kotlin
LaunchedEffect(Unit) {
    delay(2000)
    snackbarHostState.showSnackbar("Hello after 2 seconds!")
}
```

---

## 🧰 Practical Example

```kotlin
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel = viewModel()) {
    val userData by viewModel.userData.collectAsState()

    // Fetch data every time userId changes
    LaunchedEffect(userId) {
        viewModel.loadUserData(userId)
    }

    if (userData == null) {
        Text("Loading user data for $userId...")
    } else {
        Text(userData!!)
    }
}
```

---

## 🔍 Comparison with Other Side-Effect APIs

| API                        | Purpose                                                      | Key Difference                            |
| -------------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| **LaunchedEffect**         | Run suspend code tied to composition                         | Automatically manages coroutine lifecycle |
| **SideEffect**             | Run immediate, non-suspend actions after every recomposition | Does not support suspend functions        |
| **DisposableEffect**       | Run setup & cleanup when entering/leaving composition        | Ideal for listeners or external resources |
| **rememberCoroutineScope** | Gives manual control of coroutine scope                      | You manage when to start/stop coroutines  |

---

## ⚠️ Common Mistakes

| Mistake                                   | Why it’s a problem                        | Fix                                                      |
| ----------------------------------------- | ----------------------------------------- | -------------------------------------------------------- |
| Omitting the key                          | The effect may restart unnecessarily      | Use `Unit` or a stable key                               |
| Running heavy work on every recomposition | Wasteful and may cause flickering         | Use proper keys to control reruns                        |
| Calling non-suspend side effects          | `LaunchedEffect` is for suspend code only | Use `SideEffect` instead                                 |
| Forgetting lifecycle cancellation         | Coroutines may leak                       | Compose cancels automatically — use it to your advantage |

---

## 💡 Best Practices

✅ Use `LaunchedEffect(Unit)` for one-time initialization.
✅ Use meaningful keys (like `userId`, `page`, or `filter`) to control re-runs.
✅ Keep side-effects **idempotent** (safe to re-run).
✅ Avoid doing heavy computation inside it — offload to ViewModel if possible.
✅ Don’t launch coroutines in plain composable code — always use a proper side-effect API.


## rememberCoroutineScope: obtain a composition-aware scope to launch a coroutine outside a composable
As LaunchedEffect is a composable function, it can only be used inside other composable functions. In order to launch a coroutine outside of a composable, but scoped so that it will be automatically canceled once it leaves the composition, use rememberCoroutineScope. Also use rememberCoroutineScope whenever you need to control the lifecycle of one or more coroutines manually, for example, cancelling an animation when a user event happens.

# Jetpack Compose Side-Effects — Rephrased & Reformatted Guide

## What’s a side-effect?

A **side-effect** is any change to app state that happens **outside** a composable function’s scope. Because composables can recompose unpredictably, in different orders, and even discard work, composables should **ideally be pure** (no side-effects).

Sometimes you *do* need side-effects (e.g., show a snackbar once, navigate after a state change). Use Compose’s **Effect APIs** so those actions run predictably and with the right lifecycle awareness.

> **Key term**: An **effect** is a composable that doesn’t emit UI and causes side-effects to run after composition completes.

**Guidelines**

* Keep effects **UI-related** and don’t break **unidirectional data flow**.
* Compose embraces **coroutines** for async work—no callbacks required.

---

## Effect APIs at a glance

| API                      | Use it for                                                                        | Starts / Cancels           | Restarts when keys change? | Gotchas                                         |
| ------------------------ | --------------------------------------------------------------------------------- | -------------------------- | -------------------------- | ----------------------------------------------- |
| `LaunchedEffect`         | Run suspend work tied to a composable                                             | On enter / On leave        | ✔️                         | Don’t forget sensible keys                      |
| `rememberCoroutineScope` | Launch coroutines from callbacks (e.g., onClick) with composition-aware scope     | Scope lives with call site | n/a                        | Use to manually start/cancel work               |
| `rememberUpdatedState`   | Capture the **latest** value/lambda **without** restarting an effect              | n/a                        | n/a                        | Great for long-lived effects                    |
| `DisposableEffect`       | Work that needs **setup + cleanup**                                               | On enter / On leave        | ✔️                         | Must call `onDispose { ... }`                   |
| `SideEffect`             | Publish Compose state to **non-Compose** code after each successful recomposition | After each recomposition   | n/a                        | Don’t do work before composition succeeds       |
| `produceState`           | Turn external/non-Compose state into `State<T>`                                   | On enter / On leave        | ✔️                         | Conflates equal values (no extra recomposition) |
| `derivedStateOf`         | Reduce unnecessary recomposition when inputs change **more often** than UI needs  | n/a                        | n/a                        | Expensive—use only when needed                  |
| `snapshotFlow`           | Convert Compose `State` reads to a cold `Flow`                                    | On collect / cancel        | n/a                        | Emits distinct values only                      |

---

## `LaunchedEffect`: run suspend work in a composable’s scope

When the composable enters composition, `LaunchedEffect` starts a coroutine; it cancels when the composable leaves. If the **keys** change, the running coroutine cancels and a new one starts.

**Example: pulsing an alpha with a configurable delay**

```kotlin
// Allow the pulse rate to be adjusted (e.g., speed up when time is short)
var pulseRateMs by remember { mutableLongStateOf(3000L) }
val alpha = remember { Animatable(1f) }

LaunchedEffect(pulseRateMs) { // Restart when the pulse rate changes
    while (isActive) {
        delay(pulseRateMs)        // Wait between pulses
        alpha.animateTo(0f)
        alpha.animateTo(1f)
    }
}
```

---

## `rememberCoroutineScope`: get a composition-aware `CoroutineScope`

`LaunchedEffect` can only be used inside composables. To launch coroutines from **event handlers** or other places (but still cancel them when the caller leaves composition), use `rememberCoroutineScope()`.

**Example: show a `Snackbar` on button tap**

```kotlin
@Composable
fun MoviesScreen(snackbarHostState: SnackbarHostState) {
    // Scope is tied to MoviesScreen's lifecycle
    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = { SnackbarHost(hostState = snackbarHostState) }
    ) { contentPadding ->
        Column(Modifier.padding(contentPadding)) {
            Button(
                onClick = {
                    // Launch a coroutine for the snackbar
                    scope.launch {
                        snackbarHostState.showSnackbar("Something happened!")
                    }
                }
            ) {
                Text("Press me")
            }
        }
    }
}
```

---

## `rememberUpdatedState`: capture the latest value without restarting

If your long-lived effect should **not** restart when some value changes, wrap that value with `rememberUpdatedState` and use the **current** value inside the effect.

**Example: splash screen timeout that shouldn’t restart on recomposition**

```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {
    // Always refers to the latest onTimeout
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    // Tie to call site's lifecycle: doesn't restart on recomposition
    LaunchedEffect(true) {
        delay(SplashWaitTimeMillis)
        currentOnTimeout()
    }

    /* Landing screen content */
}
```

> ⚠️ `LaunchedEffect(true)` is like `while(true)`: valid in some cases, but pause and confirm it’s really what you want.

---

## `DisposableEffect`: effects that need cleanup

Use when you must **register** something and **unregister** it on key changes or when the composable leaves.

**Example: listen to `Lifecycle` events for analytics**

```kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit, // Send "started" event
    onStop: () -> Unit   // Send "stopped" event
) {
    // Keep the latest callbacks without restarting the effect
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    // Recreate when lifecycleOwner changes
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_START) currentOnStart()
            else if (event == Lifecycle.Event.ON_STOP) currentOnStop()
        }

        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }

    /* Home screen content */
}
```

> A `DisposableEffect` **must** include `onDispose { ... }`. Avoid an empty `onDispose`—reconsider the API if there’s nothing to clean up.

---

## `SideEffect`: publish Compose state to non-Compose code

Runs **after every successful recomposition**. Use it to update external objects that shouldn’t change until the UI has committed.

**Example: set analytics user properties**

```kotlin
@Composable
fun rememberFirebaseAnalytics(user: User): FirebaseAnalytics {
    val analytics: FirebaseAnalytics = remember { FirebaseAnalytics() }

    // After each composition, reflect the latest userType in analytics
    SideEffect {
        analytics.setUserProperty("userType", user.userType)
    }
    return analytics
}
```

⚠️ No coroutine — it’s synchronous.
Useful for external synchronization or logging.

---

## `produceState`: turn external data into Compose `State`

Launches a composition-scoped producer that updates a returned `State<T>`. Cancels on leave; **conflates** equal values (no extra recomposition). For non-suspending sources, use `awaitDispose` to unsubscribe.

**Example: load an image from the network**

```kotlin
@Composable
fun loadNetworkImage(
    url: String,
    imageRepository: ImageRepository = ImageRepository()
): State<Result<Image>> {
    // Creates State with Result.Loading as initial value.
    // Changing url or imageRepository restarts the producer.
    return produceState<Result<Image>>(initialValue = Result.Loading, url, imageRepository) {
        val image = imageRepository.load(url)
        value = if (image == null) Result.Error else Result.Success(image)
    }
}
```

**Under the hood:** `produceState` uses `remember { mutableStateOf(initial) }` and a `LaunchedEffect` to run the producer, updating `value`.

---

## `derivedStateOf`: derive cheaper, less frequent updates

If an input changes **more often** than your UI needs, wrap the computation in `derivedStateOf` (often together with `remember`) to avoid unnecessary recompositions—similar intent to `Flow.distinctUntilChanged()`.

**Correct usage (scroll-to-top button only when past first item)**

```kotlin
@Composable
fun MessageList(messages: List<Message>) {
    Box {
        val listState = rememberLazyListState()

        LazyColumn(state = listState) {
            // ...
        }

        // Recompute only when the threshold logic changes
        val showButton by remember {
            derivedStateOf { listState.firstVisibleItemIndex > 0 }
        }

        AnimatedVisibility(visible = showButton) {
            ScrollToTopButton()
        }
    }
}
```

**Incorrect usage (no benefit—updates are needed every time anyway)**

A common mistake is to assume that, when you combine two Compose state objects, you should use **derivedStateOf** because you are "deriving state". However, this is purely overhead and not required, as shown in the following snippet:

> ⚠️ Don’t do this in your project.

```kotlin
// DO NOT USE. Incorrect usage of derivedStateOf.
var firstName by remember { mutableStateOf("") }
var lastName by remember { mutableStateOf("") }

val fullNameBad by remember { derivedStateOf { "$firstName $lastName" } } // Bad
val fullNameCorrect = "$firstName $lastName" // Good
```

In this snippet, fullName needs to update just as often as `firstName` and `lastName`. Therefore, no excess recomposition is occurring, and using derivedStateOf is not necessary.

---

## `snapshotFlow`: turn `State` reads into a `Flow`

`snapshotFlow { ... }` emits the block’s result when **collected**, then emits again whenever any read `State` changes to a **new** value (distinct semantics).

**Example: send analytics after the user scrolls past the first item**

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```

---

## Restarting effects: choosing the right keys

Many effects (`LaunchedEffect`, `produceState`, `DisposableEffect`, …) accept **keys**. When a key changes, the current effect **cancels/disposes** and a new one starts.

**Rules of thumb**

* Add **all mutable values used inside the effect** as keys—unless changing them **shouldn’t** restart the effect.
* For values that **shouldn’t** trigger restarts, wrap them with `rememberUpdatedState` and read the **current** value inside the effect.
* Values created with `remember` **without keys** don’t need to be keys (they don’t change).

**Example**

```kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit,
    onStop: () -> Unit
) {
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event -> /* ... */ }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

Here, `currentOnStart`/`currentOnStop` **aren’t** keys because `rememberUpdatedState` ensures they always reflect the latest value without restarts. `lifecycleOwner` **is** a key—if it changes and you don’t restart, you’ll observe the wrong lifecycle.

---

## Constants as keys

You can pass a constant key (e.g., `true` or `Unit`) to tie an effect to the **call site’s** lifecycle. This is valid in some scenarios (like the splash example), but use sparingly and double-check it’s appropriate.

---

### Quick checklist

* Need suspend work tied to a composable? → **`LaunchedEffect(keys)`**
* Need to launch from a click/handler but cancel with the UI? → **`rememberCoroutineScope()`**
* Long-lived effect, value should update without restart? → **`rememberUpdatedState(value)`**
* Requires setup + cleanup? → **`DisposableEffect(keys)`**
* Push state to external API after recomposition? → **`SideEffect { ... }`**
* Convert external data to `State`? → **`produceState(...)`** (use `awaitDispose` for unsubscribing)
* Cut down recompositions from chatty inputs? → **`derivedStateOf { ... }`**
* Observe `State` as a `Flow`? → **`snapshotFlow { ... }`**
* Unsure about keys? Add the ones used **inside** the effect, or wrap with `rememberUpdatedState` if updates shouldn’t restart it.