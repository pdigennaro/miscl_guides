# Kotlin Coroutines vs Threads — Practical README

A hands-on guide to when and how to use Kotlin **coroutines**, how they differ from **threads**, and practical patterns you can drop into your codebase.

---

## TL;DR

- Prefer **coroutines** for almost all async or concurrent work.
- Keep work inside a **scope**; let **structured concurrency** manage lifecycle and cancellation.
- Use the right **Dispatcher**: `Dispatchers.IO` for blocking I/O, `Dispatchers.Default` for CPU-bound tasks, `Dispatchers.Main` for UI.
- Avoid `GlobalScope`, avoid blocking calls inside coroutines, and handle cancellation.

---

## Quick Start

```kotlin
// build.gradle.kts (Kotlin/JVM or Android)
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")
    // Android
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0")
}
```

```kotlin
// Simple coroutine example
import kotlinx.coroutines.*

fun main() = runBlocking { // a scope for this main
    val job = launch(Dispatchers.Default) { // start a coroutine
        repeat(3) {
            println("tick $it on ${Thread.currentThread().name}")
            delay(500) // suspends, doesn't block the thread
        }
    }
    job.join() // wait for completion
}
```

---

## Mental Model: Threads vs Coroutines

| Concept           | Threads                     | Coroutines                                             |
| ----------------- | --------------------------- | ------------------------------------------------------ |
| What they are     | OS-level execution units    | Language/runtime-level tasks that *run on* threads     |
| Cost              | Heavy (MBs stack), limited  | Lightweight (thousands fine)                           |
| Waiting           | `sleep()` blocks the thread | `delay()` suspends without blocking                    |
| Lifecycle         | Manual; must track and stop | **Structured** via scopes & jobs                       |
| Cancellation      | Cooperative & clunky        | First-class, exception-based (`CancellationException`) |
| Composition       | Callbacks, latches, futures | `suspend` functions, `async/await`, flows              |
| Error propagation | Detached; easy to lose      | Bubbles through scope, supervisable                    |

---

## Core Building Blocks

### Scopes & Structured Concurrency

A **CoroutineScope** owns children coroutines. When the scope is canceled, so are its children.

```kotlin
class Repo(private val external: ExternalApi) : CoroutineScope by CoroutineScope(SupervisorJob() + Dispatchers.IO) {
    suspend fun fetch(): Data = coroutineScope { // structured child scope
        val a = async { external.loadA() }
        val b = async { external.loadB() }
        combine(a.await(), b.await())
    }
    fun close() { cancel() } // cancel all ongoing work
}
```

### Dispatchers

- `Dispatchers.Main`: UI thread (Android/Compose/desktop UI)
- `Dispatchers.IO`: Offload blocking I/O (disk, network, DB)
- `Dispatchers.Default`: CPU-bound parallelism
- `newSingleThreadContext("Name")`: rare; for confinement to one thread

```kotlin
suspend fun loadFile(path: String): String = withContext(Dispatchers.IO) {
    java.io.File(path).readText() // OK to block here
}
```

### Launch vs Async

- `launch {}` → fire-and-forget, returns `Job`.
- `async {}` → returns `Deferred<T>`; call `await()` to get the result.

```kotlin
suspend fun computeBoth(): Int = coroutineScope {
    val x = async { computeX() }
    val y = async { computeY() }
    x.await() + y.await()
}
```

### Cancellation

Cancellation is cooperative: check regularly (suspension points do this).

```kotlin
suspend fun poll() {
    while (currentCoroutineContext().isActive) {
        doWorkChunk()
        delay(100)
    }
}
```

Make blocking code cancellable by switching context:

```kotlin
suspend fun <T> blockingCall(fn: () -> T): T = withContext(Dispatchers.IO) { fn() }
```

### Exception Handling

```kotlin
val handler = CoroutineExceptionHandler { _, e -> log(e) }

scope.launch(handler) {
    try {
        risky()
    } catch (e: CancellationException) {
        throw e // never swallow cancellation
    } catch (e: Throwable) {
        log("Recovered: $e")
    }
}
```

`SupervisorJob()` prevents one child’s failure from canceling siblings.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
```

---

## Common Patterns

### 1) Parallel I/O fan-out/fan-in

```kotlin
suspend fun loadUserHome(userId: Id) = coroutineScope {
    val user = async(Dispatchers.IO) { api.user(userId) }
    val posts = async(Dispatchers.IO) { api.posts(userId) }
    val friends = async(Dispatchers.IO) { api.friends(userId) }
    Triple(user.await(), posts.await(), friends.await())
}
```

### 2) Timeouts & Retry

```kotlin
suspend fun <T> withRetries(times: Int = 3, block: suspend () -> T): T {
    var last: Throwable? = null
    repeat(times) {
        try { return block() } catch (t: Throwable) { last = t; delay(200L * (it + 1)) }
    }
    throw last ?: IllegalStateException()
}

suspend fun fetchWithTimeout(): Data = withTimeout(2_000) { api.fetch() }
```

### 3) UI: ViewModel scope (Android)

```kotlin
class MyViewModel : ViewModel() {
    val state = MutableStateFlow(UiState())

    fun load() = viewModelScope.launch {
        state.value = state.value.copy(loading = true)
        state.value = state.value.copy(data = repo.fetch(), loading = false)
    }
}
```

### 4) Channels (hot) vs Flow (cold)

- **Channel**: push-style, backpressure via suspension.
- **Flow**: cold, declarative stream; supports operators, cancellation, and backpressure naturally.

```kotlin
// Channel example
val channel = Channel<Event>(capacity = Channel.BUFFERED)
scope.launch { for (e in channel) handle(e) }
scope.launch { channel.send(Event.Click) }

// Flow example
fun numbers(): Flow<Int> = flow {
    for (i in 1..3) { emit(i); delay(100) }
}

suspend fun use() { numbers().map { it * 2 }.collect { println(it) } }
```

---

## Migrating From Threads to Coroutines

### Typical thread-based code

```kotlin
val executor = Executors.newFixedThreadPool(4)
val future = executor.submit<Int> { heavyCompute() }
val result = future.get() // blocks
```

### Idiomatic coroutine replacement

```kotlin
suspend fun result(): Int = withContext(Dispatchers.Default) { heavyCompute() }

// or parallel
suspend fun result2(): Int = coroutineScope {
    val a = async(Dispatchers.Default) { computeA() }
    val b = async(Dispatchers.Default) { computeB() }
    a.await() + b.await()
}
```

### Interop with legacy APIs

```kotlin
suspend fun <T> Future<T>.await(): T = suspendCancellableCoroutine { cont ->
    cont.invokeOnCancellation { this.cancel(true) }
    this.whenComplete { value, throwable ->
        if (throwable != null) cont.resumeWithException(throwable)
        else cont.resume(value) {}
    }
}
```

---

## Testing Coroutines

```kotlin
// build.gradle.kts
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")
```

```kotlin
import kotlinx.coroutines.test.runTest

@Test
fun loads_data() = runTest {
    val repo = FakeRepo()
    val result = repo.fetch()
    assertEquals(expected, result)
}
```

Use `runTest` for deterministic virtual time and automatic job cleanup.

---

## Performance Notes

- Coroutines remove **idle** blocking; they don’t make CPU-bound code magically faster. Use `Dispatchers.Default` and algorithmic improvements.
- Prefer batching I/O and minimizing context switches.
- Beware excessive `async`—it has overhead. Use sequential `withContext` when dependencies exist.

---

## Common Pitfalls & How to Avoid Them

- ❌ **Blocking inside coroutines** (`Thread.sleep`, long CPU work on Main) → ✅ Use `delay`, and switch to `Dispatchers.IO`/`Default` via `withContext`.
- ❌ \*\*Leaking coroutines with \*\*\`\` → ✅ Tie to a lifecycle scope (`viewModelScope`, `lifecycleScope`, custom `CoroutineScope`).
- ❌ \*\*Swallowing \*\*\`\` → ✅ Rethrow; it signals cooperative cancellation.
- ❌ \*\*Forgetting exception handlers on top-level \*\*\`\` → ✅ Attach `CoroutineExceptionHandler` or use `supervisorScope` when appropriate.
- ❌ **Sharing mutable state across threads** → ✅ Use confinement (single-thread context), immutable data, or `Mutex`/`StateFlow`.

---

## Mini Cheatsheet

```kotlin
// Create a scope
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

// Fire-and-forget
val job = scope.launch { doStuff() }

// Get a result
val value = scope.async { compute() }.await()

// Switch threads
withContext(Dispatchers.IO) { blockingOp() }

// Timeout
withTimeout(1500) { slowOp() }

// Handle exceptions
val handler = CoroutineExceptionHandler { _, e -> log(e) }
scope.launch(handler) { risky() }

// Cancel
job.cancel()

// Flow
flowOf(1,2,3).map { it * 2 }.collect { println(it) }
```

---

## When Threads Are Still OK

- Integrating with libraries that **require** thread primitives (custom `Thread`, `ExecutorService`).
- Real-time or low-level scenarios needing strict thread affinity.
- Concurrency primitives demos/tests or extremely simple background work (but coroutines still fine).

If you start with threads, you’ll often re-implement scheduling, cancellation, and error propagation—coroutines give you these out of the box.

---

## FAQ

**Q: Do coroutines run in parallel?**\
*A:* They can, depending on dispatcher and availability of threads. They’re concurrent by default; parallelism happens on multi-threaded dispatchers.

**Q: Do I need ****\`\`**** for every call?**\
*A:* No. Use `async` only when you need parallel results. Otherwise write straight-line `suspend` code.

**Q: How many coroutines is too many?**\
*A:* Thousands are fine; use backpressure (Flow/Channel) and avoid spawning unbounded work.

**Q: What about thread-locals?**\
*A:* Use `ThreadContextElement` (e.g., `MDCContext`) or pass data explicitly.

---

## Further Reading / Keywords

- Structured concurrency
- CoroutineScope, Job, SupervisorJob
- Dispatchers and context
- Flows, Channels, backpressure
- Cancellation & timeouts
- Coroutine testing (`runTest`)

---

*Happy suspending!*

