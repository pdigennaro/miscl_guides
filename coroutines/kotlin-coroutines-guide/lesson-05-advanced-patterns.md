# Lesson 5: Advanced Coroutines Patterns and Best Practices

## 1. Custom Coroutine Builders

**Creating your own `retry` builder**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun <T> Flow<T>.retry(
    times: Int = Int.MAX_VALUE,
    delayMs: Long = 1000L,
    predicate: suspend (Throwable) -> Boolean = { true }
): Flow<T> = flow {
    var currentDelay = delayMs
    repeat(times - 1) { attempt ->
        try {
            collect { value -> emit(value) }
            return@flow  // Success, exit
        } catch (e: Throwable) {
            if (!predicate(e)) throw e
            if (attempt < times - 1) {
                delay(currentDelay)
                currentDelay *= 2  // Exponential backoff
            } else {
                throw e
            }
        }
    }
    // Final attempt
    collect { value -> emit(value) }
}

// Usage
fun main() = runBlocking {
    val unreliableFlow = flow {
        var attempts = 0
        emit(1)
        delay(100L)
        emit(2)
        throw RuntimeException("Simulated failure")
        emit(3)
    }
    
    unreliableFlow
        .retry(times = 3, delayMs = 500L) { error ->
            println("Retrying after error: ${error.message}")
            true
        }
        .catch { e ->
            println("Final error: ${e.message}")
            emit(-1)
        }
        .collect { value ->
            println("Received: $value")
        }
}
```

## 2. Coroutine Scoping Patterns

**Repository Pattern with Scopes**

```kotlin
import kotlinx.coroutines.*

class UserRepository {
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private val userCache = mutableMapOf<Int, User>()
    
    suspend fun getUser(id: Int): User? {
        return withContext(Dispatchers.IO) {
            // Check cache first
            userCache[id] ?: fetchUserFromNetwork(id)
        }
    }
    
    private suspend fun fetchUserFromNetwork(id: Int): User? {
        delay(1000L)  // Simulate network
        val user = User(id, "User $id")
        userCache[id] = user
        return user
    }
    
    fun cleanup() {
        scope.cancel()
    }
}

data class User(val id: Int, val name: String)

class UserViewModel(private val repository: UserRepository) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
    
    fun loadUser(id: Int) {
        scope.launch {
            try {
                val user = repository.getUser(id)
                user?.let { displayUser(it) }
            } catch (e: Exception) {
                showError(e.message ?: "Unknown error")
            }
        }
    }
    
    private fun displayUser(user: User) {
        println("Displaying: ${user.name}")
    }
    
    private fun showError(message: String) {
        println("Error: $message")
    }
    
    fun onCleared() {
        scope.cancel()
        repository.cleanup()
    }
}
```

## 3. Channel Communication

**Producer-Consumer Pattern**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    // Create a channel
    val channel = Channel<Int>(capacity = 4)  // Buffer size 4
    
    // Producer
    val producer = launch {
        for (i in 1..10) {
            println("Producing: $i")
            channel.send(i)  // Will block if buffer is full
            delay(200L)
        }
        channel.close()  // Close the channel
    }
    
    // Consumer
    val consumer = launch {
        for (value in channel) {  // Receive all values
            println("Consuming: $value")
            delay(500L)
        }
    }
    
    producer.join()
    consumer.join()
}
```

**Channel Types and Capacity**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    // Unlimited capacity
    val unlimited = Channel<String>(Channel.UNLIMITED)
    
    // Buffered (fixed size)
    val buffered = Channel<String>(capacity = 3)
    
    // Rendezvous (no buffer, sender and receiver must meet)
    val rendezvous = Channel<String>()
    
    // Conflated (keep only latest value)
    val conflated = Channel<String>(capacity = Channel.CONFLATED)
    
    println("Unlimited buffer: ${unlimited.isClosedForSend}")
    println("Buffered buffer size: ${buffered.isBuffered}")
    println("Conflated: ${conflated.isBufferOverflow}")
}
```

## 4. Select Expression

**Waiting for Multiple Operations**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*

fun main() = runBlocking {
    val channel1 = Channel<String>()
    val channel2 = Channel<String>()
    val timeoutChannel = Channel<String>()
    
    // Launch producers
    launch {
        delay(1000L)
        channel1.send("From channel 1")
    }
    
    launch {
        delay(2000L)
        channel2.send("From channel 2")
    }
    
    launch {
        delay(3000L)
        timeoutChannel.send("Timeout")
    }
    
    // Select first available result
    val result = select<String> {
        channel1.onReceive { it }
        channel2.onReceive { it }
        timeoutChannel.onReceive { it }
    }
    
    println("First result: $result")
    
    // Close remaining channels
    channel1.close()
    channel2.close()
    timeoutChannel.close()
}

// Output:
// First result: From channel 1
```

**Select with Promises (Deferred)**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*

fun main() = runBlocking {
    val deferred1 = async { delay(1000L); "Result 1" }
    val deferred2 = async { delay(2000L); "Result 2" }
    val deferred3 = async { delay(500L); "Result 3" }
    
    val firstResult = select<String> {
        deferred1.onAwait { it }
        deferred2.onAwait { it }
        deferred3.onAwait { it }
    }
    
    println("First available: $firstResult")
    
    // Cancel remaining
    deferred1.cancel()
    deferred2.cancel()
    deferred3.cancel()
}

// Output:
// First available: Result 3
```

## 5. Thread-Safe Data Structures

**Atomic Operations with kotlinx.atomicfu**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicReference

class Counter {
    private val count = AtomicInteger(0)
    private val history = AtomicReference<List<String>>(emptyList())
    
    suspend fun increment() {
        count.incrementAndGet()
        history.updateAndGet { 
            (it + "Increment").toMutableList() 
        }
    }
    
    fun getCount(): Int = count.get()
    fun getHistory(): List<String> = history.get()
}

fun main() = runBlocking {
    val counter = Counter()
    
    val jobs = List(100) { index ->
        launch {
            repeat(100) {
                counter.increment()
            }
        }
    }
    
    jobs.joinAll()
    
    println("Final count: ${counter.getCount()}")
    println("Expected: 10000")
    println("History size: ${counter.getHistory().size}")
}
```

## 6. Flow Processing Patterns

**Debouncing and Throttling**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun <T> Flow<T>.debounce(timeoutMillis: Long): Flow<T> = flow {
    var lastEmissionTime = 0L
    collect { value ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastEmissionTime >= timeoutMillis) {
            emit(value)
            lastEmissionTime = currentTime
        }
    }
}

fun <T> Flow<T>.throttleFirst(windowDuration: Long): Flow<T> = flow {
    var lastEmissionTime = 0L
    collect { value ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastEmissionTime >= windowDuration) {
            emit(value)
            lastEmissionTime = currentTime
        }
    }
}

fun main() = runBlocking {
    (1..10).asFlow()
        .onEach { delay(100L) }  // Emit every 100ms
        .debounce(250L)          // Wait 250ms between emissions
        .collect { value ->
            println("Debounced: $value at ${System.currentTimeMillis() % 10000}")
        }
}
```

**Buffering and Windowing**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    // Buffering
    val buffered = flow {
        repeat(5) { i ->
            delay(100L)
            println("Producing: $i")
            emit(i)
        }
    }
    .buffer(2)  // Buffer size 2
    
    buffered.collect { value ->
        delay(200L)  // Consume slower
        println("Consumed: $value")
    }
    
    println("---")
    
    // Windowing (group by time)
    val windowed = flow {
        repeat(10) { i ->
            delay(300L)
            emit(i)
        }
    }
    .windowed(size = 3, step = 1, inclusive = true)
    
    windowed.collect { window ->
        println("Window: $window")
    }
}
```

## 7. Memory Management Patterns

**WeakReference for Caching**

```kotlin
import kotlinx.coroutines.*
import java.lang.ref.WeakReference

class SmartCache {
    private val cache = mutableMapOf<String, WeakReference<String>>()
    
    suspend fun getOrCompute(key: String, compute: suspend () -> String): String {
        val cached = cache[key]?.get()
        if (cached != null) {
            println("Cache hit for $key")
            return cached
        }
        
        println("Computing for $key")
        val result = compute()
        cache[key] = WeakReference(result)
        return result
    }
}

fun main() = runBlocking {
    val cache = SmartCache()
    
    println(cache.getOrCompute("key1") { delay(1000L); "Value 1" })
    println(cache.getOrCompute("key1") { delay(1000L); "Value 1 (cached)" })
    println(cache.getOrCompute("key2") { delay(500L); "Value 2" })
}
```

## 8. Structured Concurrency Patterns

**Factory Pattern for Coroutine Scopes**

```kotlin
import kotlinx.coroutines.*

class ScopeFactory {
    companion object {
        fun createMainScope(): CoroutineScope {
            return CoroutineScope(Dispatchers.Main + SupervisorJob() + CoroutineName("MainScope"))
        }
        
        fun createIOScope(): CoroutineScope {
            return CoroutineScope(Dispatchers.IO + SupervisorJob() + CoroutineName("IOScope"))
        }
        
        fun createCustomScope(
            name: String,
            dispatcher: CoroutineDispatcher
        ): CoroutineScope {
            return CoroutineScope(dispatcher + SupervisorJob() + CoroutineName(name))
        }
    }
}

// Usage
fun main() = runBlocking {
    val mainScope = ScopeFactory.createMainScope()
    val ioScope = ScopeFactory.createIOScope()
    val customScope = ScopeFactory.createCustomScope("CustomScope", Dispatchers.Default)
    
    mainScope.launch {
        println("Running on main scope: ${coroutineContext[CoroutineName]}")
    }
    
    ioScope.launch {
        println("Running on IO scope: ${coroutineContext[CoroutineName]}")
    }
    
    customScope.launch {
        println("Running on custom scope: ${coroutineContext[CoroutineName]}")
    }
    
    delay(1000L)
}
```

## 9. Testing Patterns

**Test Coroutine Lifecycle**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*

class UserService {
    suspend fun getUser(id: Int): User {
        delay(1000L)  // Simulate network
        return User(id, "User $id")
    }
}

data class User(val id: Int, val name: String)

class UserViewModel(private val service: UserService) {
    private var userState = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = userState.asStateFlow()
    
    suspend fun loadUser(id: Int) {
        try {
            val user = service.getUser(id)
            userState.value = user
        } catch (e: Exception) {
            userState.value = null
        }
    }
}

// Test with TestScope
fun `test user loading`() = runTest {
    val service = UserService()
    val viewModel = UserViewModel(service)
    
    // Advance time manually
    val job = launch { viewModel.loadUser(1) }
    advanceTimeBy(1000L)  // Simulate the network delay
    job.join()
    
    val user = viewModel.user.value
    assertEquals(1, user?.id)
    assertEquals("User 1", user?.name)
}
```

## 10. Performance Optimization Patterns

**Connection Pooling**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import java.util.concurrent.*

class ConnectionPool(
    private val maxConnections: Int = 10,
    private val connectionFactory: suspend () -> Connection
) {
    private val pool = ArrayDeque<Connection>()
    private val semaphore = Semaphore(maxConnections)
    
    suspend fun <T> withConnection(block: suspend (Connection) -> T): T {
        semaphore.acquire()
        try {
            val connection = if (pool.isNotEmpty()) {
                pool.removeFirst()
            } else {
                connectionFactory()
            }
            
            return try {
                block(connection)
            } finally {
                pool.addLast(connection)
            }
        } finally {
            semaphore.release()
        }
    }
}

class Connection {
    fun execute(query: String): String {
        return "Result for $query"
    }
}

fun main() = runBlocking {
    val pool = ConnectionPool(maxConnections = 3) {
        delay(100L)  // Simulate connection creation
        Connection()
    }
    
    val results = List(10) { index ->
        async {
            pool.withConnection { connection ->
                connection.execute("Query $index")
            }
        }
    }
    
    results.awaitAll().forEach { println(it) }
}
```

## 11. Error Recovery Patterns

**Circuit Breaker Pattern**

```kotlin
import kotlinx.coroutines.*
import kotlin.math.pow

class CircuitBreaker(
    private val failureThreshold: Int = 5,
    private val timeoutMillis: Long = 60000L,
    private val resetTimeoutMillis: Long = 10000L
) {
    private var state = State.CLOSED
    private var failureCount = 0
    private var lastFailureTime = 0L
    
    suspend fun <T> execute(operation: suspend () -> T): T {
        when (state) {
            State.OPEN -> {
                if (System.currentTimeMillis() - lastFailureTime > resetTimeoutMillis) {
                    state = State.HALF_OPEN
                } else {
                    throw CircuitOpenException("Circuit breaker is open")
                }
            }
            else -> { /* CLOSED or HALF_OPEN */ }
        }
        
        return try {
            val result = operation()
            if (state == State.HALF_OPEN) {
                onSuccess()
            }
            result
        } catch (e: Exception) {
            onFailure()
            throw e
        }
    }
    
    private fun onSuccess() {
        state = State.CLOSED
        failureCount = 0
    }
    
    private fun onFailure() {
        failureCount++
        lastFailureTime = System.currentTimeMillis()
        
        if (failureCount >= failureThreshold) {
            state = State.OPEN
        }
    }
    
    private enum class State { CLOSED, OPEN, HALF_OPEN }
}

class CircuitOpenException(message: String) : Exception(message)

// Usage
fun main() = runBlocking {
    val circuitBreaker = CircuitBreaker()
    var attempt = 0
    
    repeat(10) { i ->
        try {
            val result = circuitBreaker.execute {
                attempt++
                if (attempt <= 3) {
                    throw RuntimeException("Service unavailable")
                }
                "Success after $attempt attempts"
            }
            println("Result: $result")
        } catch (e: Exception) {
            println("Error: ${e.message}")
        }
        delay(1000L)
    }
}
```

## Key Takeaways from Lesson 5

1. **Custom builders** - Extend coroutine functionality
2. **Repository patterns** - Proper scope management
3. **Channels** - Inter-coroutine communication
4. **Select expressions** - Wait for multiple operations
5. **Thread safety** - Use atomic operations for shared data
6. **Flow patterns** - Advanced processing techniques
7. **Memory management** - Weak references and cleanup
8. **Structured concurrency** - Factory patterns for scopes
9. **Testing** - Use TestScope for coroutine testing
10. **Performance** - Connection pooling and optimization
11. **Error recovery** - Circuit breaker pattern for resilience

## Final Practice Exercise

Build a complete application pattern that includes:
1. Repository with custom retry mechanism
2. ViewModel with proper scoping
3. Flow-based UI state management
4. Error handling with circuit breaker
5. Connection pooling for network requests
6. Comprehensive testing

<details>
<summary>Solution (Overview)</summary>

```kotlin
// Pattern Summary:
// 1. Repository: Uses flow, retry, and connection pooling
// 2. ViewModel: Uses StateFlow and proper cancellation
// 3. UI Layer: Collects flows with debouncing
// 4. Error Handling: Circuit breaker + retry
// 5. Testing: TestScope with advanceTimeBy
// 6. Performance: Connection pooling and resource management
```

This pattern provides:
- **Resilience**: Retry + circuit breaker
- **Performance**: Connection pooling + structured concurrency
- **Testability**: TestScope + proper scoping
- **User Experience**: StateFlow + debouncing
- **Maintainability**: Repository pattern + clear separation
</details>

---

Next: [Lesson 6: Real-World Applications](lesson-06-real-world.md)