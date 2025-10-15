# 🧭 Jetpack Compose Side Effects — Comprehensive Summary

---

## ⚙️ What Are Side Effects?

In Jetpack Compose, **UI = f(state)** — composables are *pure functions* of state.
But real apps need to do things *outside Compose* (like logging, fetching data, showing toasts, etc.).

Those are **side effects** — operations that affect the outside world or need to run beyond the scope of recomposition.

Compose provides **controlled effect APIs** to handle such cases safely and predictably.

---

## 🧩 The Seven Major Side Effect APIs

| API                          | Purpose                                                   | Coroutine? | Cleanup?                  | Triggered When                                | Common Use                                                   |
| ---------------------------- | --------------------------------------------------------- | ---------- | ------------------------- | --------------------------------------------- | ------------------------------------------------------------ |
| **`LaunchedEffect`**         | Run suspend functions in a composition-aware coroutine    | ✅          | Auto cancel on key change | First composition or key change               | Fetch data, navigate, delay, animate                         |
| **`rememberCoroutineScope`** | Get a scope tied to composition lifecycle                 | ✅ (manual) | ❌                         | On demand                                     | Launch coroutines on user events                             |
| **`SideEffect`**             | Run synchronous code after every recomposition            | ❌          | ❌                         | After every recomposition                     | Logging, updating external UI state                          |
| **`DisposableEffect`**       | Run setup/cleanup logic when entering/leaving composition | ❌          | ✅ manual                  | On first composition, key change, or disposal | Register/unregister listeners                                |
| **`produceState`**           | Bridge async data sources into Compose State              | ✅          | Auto cancel               | On first composition or key change            | Collect Flow, load data                                      |
| **`derivedStateOf`**         | Efficiently compute derived values from other State       | ❌          | ❌                         | When dependency changes                       | Derived calculations (e.g., total price)                     |
| **`snapshotFlow`**           | Convert Compose `State` into Flow                         | ✅          | Auto cancel               | When state snapshot changes                   | Observe state in ViewModel / Flow                            |
| **`rememberUpdatedState`**   | Keep latest value accessible inside stable effects        | ❌          | ❌                         | On recomposition                              | Stable callbacks or latest state inside long-running effects |

---

## 🔹 1. `LaunchedEffect`

Runs a coroutine **automatically** when a composable first enters composition or when its key(s) change.

```kotlin
@Composable
fun WelcomeScreen(userId: String) {
    LaunchedEffect(userId) {
        val user = fetchUser(userId) // suspend function
        println("Fetched user: $user")
    }

    Text("Welcome, $userId!")
}
```

✅ Cancels old coroutine if `userId` changes.
✅ Perfect for one-time or key-dependent tasks.

---

## 🔹 2. `rememberCoroutineScope`

Use for **imperative event-driven coroutines** (e.g., button clicks).

```kotlin
@Composable
fun DownloadButton() {
    val scope = rememberCoroutineScope()

    Button(onClick = {
        scope.launch { startDownload() }
    }) {
        Text("Download")
    }
}
```

✅ The scope is tied to the composable’s lifecycle.
❌ Doesn’t automatically restart or cancel unless the composable leaves composition.

---

## 🔹 3. `SideEffect`

Runs **after every recomposition** (once it succeeds).
Used for updating external states or calling non-suspend functions.

```kotlin
@Composable
fun LogState(value: String) {
    SideEffect {
        println("New value: $value")
    }
    Text(value)
}
```

⚠️ No coroutine — it’s synchronous.
Useful for external synchronization or logging.

---

## 🔹 4. `DisposableEffect`

Use when you need **setup and teardown logic** (register listeners, observers, etc.).

```kotlin
@Composable
fun LocationTracker(locationManager: LocationManager) {
    DisposableEffect(Unit) {
        val listener = LocationListener { location ->
            println("Location: $location")
        }
        locationManager.addListener(listener)

        onDispose {
            locationManager.removeListener(listener)
        }
    }
}
```

✅ Automatically runs cleanup when leaving composition or when keys change.

---

## 🔹 5. `produceState`

Creates and exposes a **State<T>** from async or external sources.

```kotlin
@Composable
fun UserName(userId: String): State<String> {
    return produceState(initialValue = "Loading...", userId) {
        value = fetchUserName(userId)
    }
}

@Composable
fun Profile(userId: String) {
    val name by UserName(userId)
    Text("Hello $name")
}
```

✅ Automatically launches/cancels coroutine based on composition.
✅ Used to **bridge non-Compose async data** into Compose world.

---

## 🔹 6. `derivedStateOf`

Used to derive **computed state** from other state values efficiently.

```kotlin
@Composable
fun CartSummary(items: List<CartItem>) {
    val total by remember(items) {
        derivedStateOf { items.sumOf { it.price } }
    }

    Text("Total: $${total}")
}
```

✅ Prevents unnecessary recompositions by recalculating only when input changes.

---

## 🔹 7. `snapshotFlow`

Turns Compose state changes into a **Flow** stream.

```kotlin
@Composable
fun ObserveScroll(scrollState: ScrollState) {
    LaunchedEffect(Unit) {
        snapshotFlow { scrollState.value }
            .collect { println("Scroll: $it") }
    }
}
```

✅ Great for observing Compose state from coroutines or ViewModels.

---

## 🔹 8. `rememberUpdatedState`

Keeps the **latest value or callback** accessible inside long-running effects without restarting them.

### Without it (❌ may call stale callback):

```kotlin
LaunchedEffect(Unit) {
    delay(5000)
    onTimeout() // might be old version
}
```

### With it (✅ always fresh):

```kotlin
val latestOnTimeout by rememberUpdatedState(onTimeout)

LaunchedEffect(Unit) {
    delay(5000)
    latestOnTimeout() // always current
}
```

✅ Keeps effect stable
✅ Reads latest values safely
✅ Common for callbacks, state references, or config values

---

## 🧠 Summary by Purpose

| Goal                                        | Use This                 |
| ------------------------------------------- | ------------------------ |
| Run a suspend function once / on key change | `LaunchedEffect`         |
| Launch coroutine from UI event              | `rememberCoroutineScope` |
| Sync external state after recomposition     | `SideEffect`             |
| Register / cleanup external resources       | `DisposableEffect`       |
| Convert async data → State                  | `produceState`           |
| Compute derived / memoized state            | `derivedStateOf`         |
| Convert Compose state → Flow                | `snapshotFlow`           |
| Keep latest value in long-running effect    | `rememberUpdatedState`   |

---

## 🧩 Visual Lifecycle Summary

```
 ┌────────────────────────────┐
 │ Composition Enters Screen  │
 └──────────────┬─────────────┘
                │
                ▼
        LaunchedEffect()      ← Run suspend logic once / per key
                │
                ▼
          Recomposition
                │
         ┌──────┴────────┐
         │               │
  SideEffect()       rememberUpdatedState() ← keep latest data
         │
         ▼
     DisposableEffect()  ← Cleanup on disposal / key change
```

---

## ✅ Best Practices

1. **Use `LaunchedEffect`** for automatic coroutines tied to lifecycle
2. **Use `rememberCoroutineScope`** for user-triggered actions
3. **Never run suspend functions directly in composables**
4. **Always cancel listeners in `DisposableEffect.onDispose`**
5. **Wrap changing callbacks in `rememberUpdatedState`**
6. **Use `derivedStateOf` to avoid expensive recomputation**
7. **Use `snapshotFlow` when bridging Compose → Flow**
8. **Don’t misuse keys** — keys control when effects restart!

---

## 🧾 Example Putting It All Together

```kotlin
@Composable
fun UserProfileScreen(
    userId: String,
    viewModel: UserViewModel,
    onTimeout: () -> Unit
) {
    val latestOnTimeout by rememberUpdatedState(onTimeout)

    // Fetch user when userId changes
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    val user by viewModel.user.collectAsState()

    // Derived premium state
    val isPremium by remember(user) {
        derivedStateOf { user.subscription == "premium" }
    }

    // Log recompositions
    SideEffect {
        println("UserProfile recomposed for ${user.name}")
    }

    // Handle cleanup when leaving screen
    DisposableEffect(Unit) {
        registerAnalyticsListener(userId)
        onDispose { unregisterAnalyticsListener(userId) }
    }

    // Timeout after 5 seconds
    LaunchedEffect(Unit) {
        delay(5000)
        latestOnTimeout()
    }

    Text(if (isPremium) "Welcome Premium User!" else "Welcome Free User!")
}
```

---

## 🏁 TL;DR — Ultimate Mental Model

| Effect                       | Lifecycle                       | Purpose                                |
| ---------------------------- | ------------------------------- | -------------------------------------- |
| 🌀 **LaunchedEffect**        | Automatic coroutine tied to key | One-time / key-dependent suspend logic |
| ⚡ **rememberCoroutineScope** | Manual coroutine control        | Event-driven async logic               |
| 🔁 **SideEffect**            | After recomposition             | Sync side data (non-suspend)           |
| 🧹 **DisposableEffect**      | Enter / exit composition        | Setup & cleanup                        |
| 📡 **produceState**          | Composition lifecycle           | Async data → State                     |
| 🧮 **derivedStateOf**        | On dependency change            | Derived calculations                   |
| 🌊 **snapshotFlow**          | On State change                 | State → Flow                           |
| 🔒 **rememberUpdatedState**  | On recomposition                | Stable effects + latest values         |