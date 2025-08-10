# Kotlin Coroutines: A Deep‑Dive Guide

> A practical, example‑rich reference to understand **coroutines, dispatchers, scopes, jobs, cancellation, exception handling, channels, and flows**—and how they compare to Java threads. All examples are self‑contained and heavily commented.

---

## Table of Contents
1. [What Are Coroutines?](#what-are-coroutines)
2. [Key Building Blocks](#key-building-blocks)
   - [Suspend functions](#suspend-functions)
   - [Dispatchers](#dispatchers)
   - [Jobs & Deferred](#jobs--deferred)
   - [Structured concurrency](#structured-concurrency)
3. [Scopes](#scopes)
   - [CoroutineScope](#coroutinescope)
   - [GlobalScope (when to avoid it)](#globalscope-when-to-avoid-it)
   - [SupervisorJob & supervisorScope](#supervisorjob--supervisorscope)
   - [Lifecycle/Custom scopes](#lifecyclecustom-scopes)
4. [Starting Coroutines: `launch` vs `async`](#starting-coroutines-launch-vs-async)
5. [Switching Contexts with `withContext`](#switching-contexts-with-withcontext)
6. [Cancellation](#cancellation)
7. [Exception Handling](#exception-handling)
8. [Channels](#channels)
   - [Types of channels](#types-of-channels)
   - [Producers, consumers, actors](#producers-consumers-actors)
   - [Backpressure, closing, and iteration](#backpressure-closing-and-iteration)
9. [Flows](#flows)
   - [Cold vs hot](#cold-vs-hot)
   - [Flow builders](#flow-builders)
   - [Operators (intermediate vs terminal)](#operators-intermediate-vs-terminal)
   - [Context, buffering, backpressure](#context-buffering-backpressure)
   - [`StateFlow` and `SharedFlow`](#stateflow-and-sharedflow)
   - [`callbackFlow` and bridging callbacks](#callbackflow-and-bridging-callbacks)
10. [Testing Coroutines & Flows](#testing-coroutines--flows)
11. [Performance Notes & Best Practices](#performance-notes--best-practices)
12. [Coroutines vs Java Threads](#coroutines-vs-java-threads)
13. [Interop with Java & Blocking Code](#interop-with-java--blocking-code)
14. [Cheat Sheet](#cheat-sheet)

---

## What Are Coroutines?
Coroutines are **lightweight, cooperative tasks** that can suspend without blocking a thread. They’re scheduled by the Kotlin runtime (via Dispatchers) and obey **structured concurrency** rules when launched in a scope.

- *Lightweight*: You can have tens of thousands of coroutines; each costs far less than a thread.
- *Non‑blocking*: `suspend` lets a function pause until a result is ready, freeing the thread.
- *Structured*: Children coroutines are tied to their parent scope, simplifying lifetime and cancellation.

**Hello, coroutine**:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // creates a CoroutineScope and blocks the main thread until complete
    launch { // child coroutine
        delay(500)
        println("World!")
    }
    println("Hello,")
}
```

**Output** (order may vary):
```
Hello,
World!
```

---

## Key Building Blocks

### Suspend functions
A `suspend` function can **suspend** and **resume** without blocking the underlying thread.
```kotlin
suspend fun fetchUser(id: String): User {
    delay(200) // pretend network call
    return User(id)
}
```

Use `suspend` for any operation that might wait (I/O, timers). They can only be called from another suspend function or a coroutine.

### Dispatchers
Dispatchers decide **which thread(s)** your coroutine runs on.

- `Dispatchers.Default` — CPU‑bound work (shared pool sized by CPU cores)
- `Dispatchers.IO` — I/O‑bound work (larger pool for blocking calls)
- `Dispatchers.Main` — UI thread (Android/desktop; requires dependency)
- `Dispatchers.Unconfined` — advanced/edge cases; starts in caller thread, resumes in suspender thread

```kotlin
suspend fun compute(): Int = withContext(Dispatchers.Default) {
    (1..1_000_000).sum()
}
```

### Jobs & Deferred
Every coroutine has a **Job** representing its lifecycle. `async` returns a `Deferred<T>` (a Job with a result).

```kotlin
val job: Job = scope.launch { /* do work */ }
val deferred: Deferred<Int> = scope.async { 42 }
val answer: Int = deferred.await()
```

### Structured concurrency
Scopes ensure children finish (or are cancelled) when the parent completes.
```kotlin
suspend fun parent() = coroutineScope { // cancels all children on failure
    val a = async { loadA() }
    val b = async { loadB() }
    a.await() + b.await()
}
```

---

## Scopes

### CoroutineScope
A `CoroutineScope` provides a `Job` + `CoroutineContext` (typically a Dispatcher) to launch coroutines.
```kotlin
class Repository(
    private val externalScope: CoroutineScope
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun startBackgroundSync() {
        scope.launch {
            while (isActive) { // cooperative cancellation check
                syncOnce()
                delay(5_000)
            }
        }
    }

    fun stop() = scope.cancel() // cancel all children
}
```

### GlobalScope (when to avoid it)
`GlobalScope.launch { ... }` is **detached** from any lifecycle. Prefer **structured** scopes; use `GlobalScope` only for truly process‑wide, fire‑and‑forget tasks.

### SupervisorJob & supervisorScope
Use **supervision** when you want sibling coroutines to **fail independently**.
```kotlin
suspend fun supervised() = supervisorScope {
    val c1 = launch { error("c1 fails") }
    val c2 = launch { delay(1000); println("c2 still runs") }
    // c2 is not cancelled by c1 failure
}
```

### Lifecycle/Custom scopes
- Android: `lifecycleScope`, `viewModelScope` tie to UI components.
- Server/Desktop: create your own `CoroutineScope` and manage its `Job`.

---

## Starting Coroutines: `launch` vs `async`

- `launch` → returns `Job` (no result). Use for side‑effects.
- `async` → returns `Deferred<T>`. Use when you **need a result** and plan to `await()` it.

```kotlin
suspend fun loadUserAndPosts(): Pair<User, List<Post>> = coroutineScope {
    val user = async { api.user() }
    val posts = async { api.posts() }
    user.await() to posts.await()
}
```

**Pitfall**: Don’t forget to `await()` a `Deferred`; otherwise work may be cancelled when scope ends.

---

## Switching Contexts with `withContext`
Use `withContext` to **move work** to another dispatcher (e.g., CPU → I/O).
```kotlin
suspend fun loadAndParse(file: Path): Json = withContext(Dispatchers.IO) {
    val text = Files.readString(file)
    withContext(Dispatchers.Default) { parseJson(text) }
}
```

---

## Cancellation
Coroutines are **cooperatively** cancellable.

- Cancelling a **Job** cancels its children.
- Cancellation throws `CancellationException` in a cancelling coroutine.
- Use `isActive`, `ensureActive()`, or cancellable suspensions (`delay`, `channel.receive`) to cooperate.

```kotlin
suspend fun pollingLoop() = coroutineScope {
    val job = launch {
        while (isActive) {
            doWorkChunk()
            delay(100) // suspension point, checks for cancellation
        }
    }
    delay(500)
    job.cancelAndJoin() // cancel & wait for cleanup
}
```

**Finally blocks** run on cancellation; use `withContext(NonCancellable)` for mandatory cleanup.
```kotlin
suspend fun writeSafely(channel: SendChannel<Int>) {
    try {
        channel.send(1)
    } finally {
        withContext(NonCancellable) {
            channel.close() // ensure resource release even if cancelled
        }
    }
}
```

---

## Exception Handling
- In **structured** scopes (`coroutineScope`), one child failure cancels siblings.
- In **supervisor** scopes, siblings continue.
- Uncaught exceptions in a `launch` coroutine crash the parent scope; in `async`, exceptions are **deferred until `await()`**.
- Use `CoroutineExceptionHandler` at the **top** of a scope to log or convert errors.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    println("Caught: $e")
}

fun main() = runBlocking(handler) {
    val job = launch { error("boom") } // handled by parent handler
    job.join()
}
```

**Try/catch in coroutines** works like normal; wrap the **suspending call** that may fail.
```kotlin
suspend fun safeCall(): Result<Data> = try {
    Result.success(api.fetch())
} catch (e: IOException) {
    Result.failure(e)
}
```

---

## Channels
Channels provide **hot**, backpressured queues for communication between coroutines (similar to Go channels).

### Types of channels
- `Channel.RENDEZVOUS` (capacity 0): sender suspends until receiver receives.
- `Channel.BUFFERED` (default, small buffer): sender suspends when buffer is full.
- `Channel.CONFLATED`: keeps only the **latest** value; older values are dropped.
- `Channel.UNLIMITED`: no suspension on send (risk of OOM if producers outpace consumers).

```kotlin
val ch = Channel<Int>(capacity = Channel.BUFFERED)
```

### Producers, consumers, actors
```kotlin
// Producer: emits numbers 1..N
fun CoroutineScope.produceInts(n: Int) = produce<Int> {
    for (i in 1..n) send(i)
}

// Consumer: squares and prints
suspend fun consumeSquares(input: ReceiveChannel<Int>) {
    for (i in input) println(i * i)
}

fun main() = runBlocking {
    val ints = produceInts(5)
    consumeSquares(ints)
    ints.cancel() // cancel producer
}
```

**Actor** pattern (single‑threaded state):
```kotlin
sealed class Msg
class Inc(val n: Int) : Msg()
class Get(val reply: CompletableDeferred<Int>) : Msg()

fun CoroutineScope.counterActor() = actor<Msg> {
    var count = 0
    for (msg in channel) when (msg) {
        is Inc -> count += msg.n
        is Get -> msg.reply.complete(count)
    }
}

fun main() = runBlocking {
    val counter = counterActor()
    counter.send(Inc(2))
    counter.send(Inc(3))
    val reply = CompletableDeferred<Int>()
    counter.send(Get(reply))
    println("count=\${reply.await()}")
    counter.close()
}
```

### Backpressure, closing, and iteration
- **Backpressure** is built‑in; `send` suspends when capacity is full.
- Close with `channel.close()`; receivers see completion of the `for (x in channel)` loop.
- Prefer `Flow` for streams of data from a single producer; use **channels** for **fan‑in/fan‑out** or multi‑producer scenarios.

---

## Flows
`Flow<T>` is a **cold, asynchronous stream** that emits values sequentially and supports operators (map, filter, etc.).

### Cold vs hot
- **Cold** (`flow {}`): starts emitting **for each collector**.
- **Hot** (`SharedFlow`, `StateFlow`): emits regardless of observers; collectors receive according to replay/buffer rules.

### Flow builders
```kotlin
val f1: Flow<Int> = flow {
    emit(1)
    delay(100)
    emit(2)
}

val f2 = flowOf(1, 2, 3)
val f3 = (1..5).asFlow()
```

### Operators (intermediate vs terminal)
- **Intermediate**: `map`, `filter`, `transform`, `debounce`, `buffer`, `conflate`, `flowOn` (returns a Flow)
- **Terminal**: `collect`, `toList`, `first`, `single`, `launchIn` (starts collection)

```kotlin
suspend fun demo(flow: Flow<Int>) {
    flow
        .map { it * it }
        .filter { it % 2 == 0 }
        .collect { println(it) }
}
```

### Context, buffering, backpressure
Use `flowOn` to change **emission** context; collection stays in the caller context.
```kotlin
val processed: Flow<Item> = source
    .map { heavy(it) } // runs where collected by default
    .flowOn(Dispatchers.Default) // move emission upstream
    .buffer(capacity = 64)       // allow upstream to run ahead
    .conflate()                  // drop intermediate values when slow
```

### `StateFlow` and `SharedFlow`
- `StateFlow<T>`: hot, **state holder** with a current value; always emits the latest value to new collectors.
- `SharedFlow<T>`: hot, general **multicast** flow with configurable replay and buffer.

```kotlin
class ViewModel {
    private val _state = MutableStateFlow(Loading)
    val state: StateFlow<UiState> = _state

    fun load() = CoroutineScope(Dispatchers.IO).launch {
        _state.value = Loading
        _state.value = dataSource.fetch()
    }
}
```

**SharedFlow example** (event bus):
```kotlin
val events = MutableSharedFlow<Event>(replay = 0, extraBufferCapacity = 16)

suspend fun publish(e: Event) { events.emit(e) }
fun observe(scope: CoroutineScope) {
    events.onEach { println("event: $it") }.launchIn(scope)
}
```

### `callbackFlow` and bridging callbacks
Wrap callback‑based APIs as flows with `callbackFlow` and **close on cancellation**.
```kotlin
fun locationFlow(): Flow<Location> = callbackFlow {
    val listener = object : LocationListener {
        override fun onUpdate(loc: Location) { trySend(loc).isSuccess }
        override fun onError(e: Throwable) { close(e) }
    }
    startLocationUpdates(listener)
    awaitClose { stopLocationUpdates(listener) } // called on collector cancellation
}
```

---

## Testing Coroutines & Flows
Use the `kotlinx-coroutines-test` module.

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.UnconfinedTestDispatcher

class RepoTest {
    @Test fun loadsInParallel() = runTest {
        val dispatcher = UnconfinedTestDispatcher(testScheduler)
        val repo = Repo(CoroutineScope(dispatcher))
        val result = repo.loadUserAndPosts()
        assertEquals(expected, result)
    }
}
```

For flows:
```kotlin
@Test fun emitsInOrder() = runTest {
    val f = flow {
        emit(1); delay(10); emit(2)
    }
    val values = f.toList()
    assertEquals(listOf(1,2), values)
}
```

---

## Performance Notes & Best Practices
- Prefer **`withContext(Dispatchers.IO)`** to call blocking I/O; don’t block `Default`/`Main`.
- Use **structured concurrency**; avoid `GlobalScope` except for process‑wide background tasks.
- Prefer **`Flow`/`StateFlow`** for streams and observable state; use **channels** for multi‑producer coordination.
- Use **timeouts**: `withTimeout` / `withTimeoutOrNull`.
- For shared mutable state, prefer `Mutex` or **actor** pattern over `synchronized`.
- Clean up: cancel scopes in `close()/onCleared()/shutdown hooks`.

---

## Coroutines vs Java Threads

### Conceptual differences
| Aspect | Coroutines | Java Threads |
|---|---|---|
| Cost | **Very low**; thousands are cheap | **High**; each thread ~MBs of stack |
| Scheduling | Managed by coroutine dispatcher; can suspend without blocking | OS/VM scheduler; blocking often required |
| Concurrency model | **Structured**, parent/child lifetimes | **Unstructured**; manual tracking |
| Cancellation | **Cooperative** (`isActive`, suspension points) | **Interruption** (`Thread.interrupt`), error‑prone |
| Communication | `Flow`, `Channel`, `Mutex`, `actor` | Shared memory, `synchronized`, `Lock`, `BlockingQueue` |
| Error Propagation | Bubbles through scope; supervision available | Uncaught exceptions handled per‑thread; no structure |

### Code side‑by‑side
**Parallel fetch (coroutines)**:
```kotlin
suspend fun fetchBoth(): Pair<A, B> = coroutineScope {
    val a = async(Dispatchers.IO) { fetchA() }
    val b = async(Dispatchers.IO) { fetchB() }
    a.await() to b.await()
}
```

**Parallel fetch (Java threads)**:
```java
ExecutorService es = Executors.newFixedThreadPool(2);
Future<A> fa = es.submit(() -> fetchA());
Future<B> fb = es.submit(() -> fetchB());
A a = fa.get(); // blocking
B b = fb.get(); // blocking
es.shutdown();
return new Pair<>(a, b);
```

**Cancellation**
```kotlin
val job = scope.launch { doWork() }
// later
job.cancel() // cooperative
```
```java
Thread t = new Thread(() -> doWork());
t.start();
// later
t.interrupt(); // may or may not stop work, depends on checks
```

---

## Interop with Java & Blocking Code

### Calling blocking APIs safely
Wrap with `withContext(Dispatchers.IO)`:
```kotlin
suspend fun readFileSafe(path: Path): String = withContext(Dispatchers.IO) {
    Files.readString(path) // blocking OK on IO pool
}
```

### Using existing executors
```kotlin
val myDispatcher = Executors.newFixedThreadPool(4).asCoroutineDispatcher()
CoroutineScope(myDispatcher).launch { /* ... */ }
```

### Bridging futures
```kotlin
suspend fun <T> Future<T>.await(): T = suspendCancellableCoroutine { cont ->
    cont.invokeOnCancellation { this.cancel(true) }
    Executors.newSingleThreadExecutor().submit {
        try { cont.resume(get()) } catch (e: Throwable) { cont.resumeWithException(e) }
    }
}
```

---

## Cheat Sheet
- **Start**: `launch{}` (no result), `async{}`/`await()` (result)
- **Scope**: `coroutineScope{}`, `supervisorScope{}`, custom `CoroutineScope(Job+Dispatcher)`
- **Context switch**: `withContext(Dispatcher)`
- **Cancel**: `job.cancel()`, check `isActive`, use cancellable suspensions
- **Exceptions**: `try/catch`, `CoroutineExceptionHandler`, supervision
- **Channels**: `Channel(capacity)`, `send/receive`, `close`, `actor`, `produce`
- **Flows**: `flow{}`, `map`, `filter`, `buffer`, `conflate`, `flowOn`, `collect`;
  hot: `StateFlow` (`.value`), `SharedFlow` (`emit`, `launchIn`)
- **Testing**: `runTest {}`, virtual time, `UnconfinedTestDispatcher`

---

## Appendix: Full, Commented Examples

### 1) End‑to‑end: network + caching + UI state with Flow
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

sealed interface UiState {
    data object Idle : UiState
    data object Loading : UiState
    data class Success(val data: List<Item>) : UiState
    data class Error(val message: String) : UiState
}

class Repo(
    private val api: Api,
    private val cache: Cache,
    private val scope: CoroutineScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
) {
    private val _state = MutableStateFlow<UiState>(UiState.Idle)
    val state: StateFlow<UiState> = _state

    fun refresh() {
        scope.launch {
            _state.value = UiState.Loading
            val result = runCatching {
                val (net, local) = coroutineScope {
                    val net = async { api.fetchItems() } // IO
                    val local = async { cache.loadItems() } // IO
                    net.await() to local.await()
                }
                cache.saveItems(net)
                (local + net).distinct()
            }
            _state.value = result.fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message ?: "unknown error") }
            )
        }
    }

    fun close() { scope.cancel() }
}
```

### 2) Channel fan‑in/fan‑out pipeline
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun CoroutineScope.producers(n: Int): List<ReceiveChannel<Int>> = List(n) {
    produce {
        repeat(10) {
            delay(50)
            send((0..100).random())
        }
    }
}

fun CoroutineScope.merger(inputs: List<ReceiveChannel<Int>>): ReceiveChannel<Int> = produce {
    val jobs = inputs.map { input ->
        launch { for (x in input) send(x) }
    }
    jobs.joinAll()
}

fun main() = runBlocking {
    val inputs = producers(3)
    val merged = merger(inputs)
    val squares = produce {
        for (x in merged) send(x * x)
    }
    repeat(5) { println(squares.receive()) }
    coroutineContext.cancelChildren()
}
```

### 3) `callbackFlow` wrapping a listener API
```kotlin
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow

interface SensorListener { fun onValue(v: Float); fun onStop() }

fun sensorValues(): Flow<Float> = callbackFlow {
    val listener = object : SensorListener {
        override fun onValue(v: Float) { trySend(v).isSuccess }
        override fun onStop() { close() }
    }
    SensorApi.register(listener)
    awaitClose { SensorApi.unregister(listener) }
}
```

---

## Further Reading
- `kotlinx.coroutines` reference docs and guides
- Kotlin language reference: coroutines and flow
- Android guides (if building for Android): lifecycle scopes, Flow with UI

*End of guide.*

