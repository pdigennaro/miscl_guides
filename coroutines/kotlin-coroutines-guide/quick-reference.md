# Quick Reference Guide

## Essential Imports
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.test.* // For testing
```

## Coroutine Builders
- `launch` - Fire and forget (returns Job)
- `async` - Returns a result (returns Deferred)
- `runBlocking` - Blocks the thread (main and tests)
- `produce` (experimental) - Creates a channel-based producer

## Dispatchers
- `Dispatchers.Default` - CPU-intensive work
- `Dispatchers.IO` - I/O operations (network, files, DB)
- `Dispatchers.Main` - Main/UI thread (Android, JavaFX)
- `Dispatchers.Unconfined` - No specific thread constraint

## Flow Operators

### Creation
- `flowOf(...)` - Create from values
- `asFlow()` - Convert collection to flow
- `flow { ... }` - Custom flow builder

### Terminal Operations
- `collect { }` - Consume all values
- `toList()`, `toSet()` - Collect to collection
- `first()` - Get first value
- `count()` - Count elements

### Intermediate Operations
- `map { }` - Transform values
- `filter { }` - Filter values
- `take(n)` - Take first n elements
- `drop(n)` - Skip first n elements
- `distinctUntilChanged()` - Remove consecutive duplicates
- `debounce(timeout)` - Delay emission
- `flatMapConcat { }` - Transform and concatenate
- `flatMapMerge(concurrency) { }` - Transform and merge concurrently

## Exception Handling
- `try-catch` blocks in coroutines
- `catch { }` operator for flows
- `supervisorScope` for independent children
- `SupervisorJob` for failure isolation
- `CoroutineExceptionHandler` for global handling

## Cancellation
- `job.cancel()` - Request cancellation
- `job.join()` - Wait for completion
- `job.cancelAndJoin()` - Cancel and wait
- `isActive` - Check if coroutine is active
- `ensureActive()` - Throw if cancelled
- `withTimeout()` / `withTimeoutOrNull()` - Automatic timeouts

## State Management
- `MutableStateFlow` - Mutable state holder
- `StateFlow` - Read-only state holder
- `MutableSharedFlow` - Mutable event bus
- `SharedFlow` - Read-only event bus

## Common Patterns

### Scope creation
```kotlin
val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
```

### Proper cleanup
```kotlin
fun cleanup() {
    scope.cancel()
}
```

### Error handling with retry
```kotlin
suspend fun robustOperation(): String = withContext(Dispatchers.IO) {
    retry(times = 3, delay = 1000L) {
        apiCall()
    }
}
```

### Flow with error handling
```kotlin
someFlow
    .catch { e -> 
        // Handle error
        emit(defaultValue)
    }
    .retry(times = 3) { e ->
        // Retry condition
        e is NetworkException
    }
    .collect { value ->
        // Process
    }
```

### Parallel operations
```kotlin
val (result1, result2, result3) = awaitAll(
    async { operation1() },
    async { operation2() },
    async { operation3() }
)
```

### Timeout operation
```kotlin
val result = withTimeoutOrNull(5000L) {
    longRunningOperation()
}
```

### Channel communication
```kotlin
val channel = Channel<String>(capacity = 3)

launch {
    channel.send("Message 1")
    channel.send("Message 2")
    channel.close() // Important: close when done
}

launch {
    for (message in channel) {
        println("Received: $message")
    }
}
```

### Select expressions
```kotlin
val result = select<String> {
    channel1.onReceive { it }
    channel2.onReceive { it }
    timeoutChannel.onReceive { it }
}
```

## Testing Utilities
- `runTest` - Test coroutines without real delays
- `TestScope` - Scoped testing environment
- `advanceTimeBy()` - Simulate time passage
- `pauseDispatcher()` / `resumeDispatcher()` - Control timing

## Best Practices

### 1. Use Structured Concurrency
```kotlin
// ✅ Good
fun main() = runBlocking {
    val data1 = async { fetchData1() }
    val data2 = async { fetchData2() }
    processData(data1.await(), data2.await())
}

// ❌ Bad - Memory leak risk
GlobalScope.launch {
    // This coroutine has no structured scope
}
```

### 2. Choose Right Dispatcher
```kotlin
// ✅ CPU-intensive work
withContext(Dispatchers.Default) {
    heavyComputation()
}

// ✅ I/O operations
withContext(Dispatchers.IO) {
    networkCall()
    fileOperations()
}
```

### 3. Handle Errors Properly
```kotlin
// ✅ Use try-catch with coroutines
launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        // Handle gracefully
    }
}

// ✅ Use supervisor for independent children
supervisorScope {
    val child1 = launch { mightFail() }
    val child2 = launch { independentWork() }
    joinAll(child1, child2)
}
```

### 4. Use Flow for Reactive Programming
```kotlin
// ✅ Reactive data stream
val userFlow = userRepository.getUsers()
    .map { it.map { user -> user.name } }
    .distinctUntilChanged()
    .debounce(300)

userFlow.collect { names ->
    updateUI(names)
}
```

### 5. Cancel Properly
```kotlin
class ViewModel : ViewModel() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    
    // Always cancel scope when ViewModel is cleared
    override fun onCleared() {
        super.onCleared()
        scope.cancel()
    }
}
```

### 6. Use StateFlow for State
```kotlin
// ✅ Proper state management
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UserState>(UserState.Loading)
    val uiState: StateFlow<UserState> = _uiState.asStateFlow()
    
    fun loadUser() {
        viewModelScope.launch {
            _uiState.value = try {
                UserState.Success(fetchUser())
            } catch (e: Exception) {
                UserState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

## Performance Tips

### 1. Use Buffering
```kotlin
// ✅ Prevents backpressure issues
expensiveFlow()
    .buffer(64) // Buffer upstream emissions
    .collect { process(it) }
```

### 2. Limit Concurrency
```kotlin
// ✅ Control parallel processing
items.asFlow()
    .mapParallel(concurrency = 4) { item ->
        processItem(item)
    }
    .collect { result ->
        // Handle result
    }
```

### 3. Use Connection Pooling
```kotlin
// ✅ Efficient resource usage
val pool = ConnectionPool(maxConnections = 10)

suspend fun <T> withConnection(block: suspend (Connection) -> T): T {
    return pool.withConnection(block)
}
```

### 4. Implement Circuit Breaker
```kotlin
class CircuitBreaker {
    suspend fun <T> execute(operation: suspend () -> T): T {
        // Implement circuit breaker logic
        // See Lesson 5 for full implementation
    }
}
```

## Common Pitfalls

### 1. Blocking Operations
```kotlin
// ❌ Don't use Thread.sleep in coroutines
launch {
    Thread.sleep(1000L) // Blocks thread!
}

// ✅ Use delay instead
launch {
    delay(1000L) // Suspends coroutine
}
```

### 2. Forgetting to Cancel
```kotlin
// ❌ Memory leak risk
class MyClass {
    val job = GlobalScope.launch { /* infinite loop */ }
}

// ✅ Proper scoping
class MyClass {
    private val scope = CoroutineScope(Dispatchers.IO)
    private val job = scope.launch { /* work */ }
    
    fun cleanup() {
        scope.cancel()
    }
}
```

### 3. Exception Propagation
```kotlin
// ❌ Exception crashes everything
launch {
    throw RuntimeException("Error!")
}

// ✅ Handle exceptions
launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        // Handle gracefully
    }
}
```

### 4. Context Switching
```kotlin
// ❌ Inefficient - switches context unnecessarily
suspend fun inefficientWork() {
    withContext(Dispatchers.Default) {
        withContext(Dispatchers.IO) {
            withContext(Dispatchers.Default) {
                work()
            }
        }
    }
}

// ✅ Efficient - minimal context switches
suspend fun efficientWork() {
    withContext(Dispatchers.Default) {
        work()
    }
}
```

## Debugging Tips

### 1. Use CoroutineName
```kotlin
launch(CoroutineName("My Coroutine")) {
    // Debug with name in stack traces
}
```

### 2. Log Coroutine Context
```kotlin
launch {
    println("Context: $coroutineContext")
    println("Name: ${coroutineContext[CoroutineName]}")
    println("Job: ${coroutineContext[Job]}")
}
```

### 3. Check Job State
```kotlin
val job = launch { work() }
println("Is active: ${job.isActive}")
println("Is cancelled: ${job.isCancelled}")
println("Is completed: ${job.isCompleted}")
```

## Learning Path

1. **Start with basics** - Learn `launch`, `async`, `runBlocking`
2. **Understand contexts** - Master dispatchers and `withContext`
3. **Handle errors** - Exception handling and cancellation
4. **Master Flow** - Reactive programming with Flow
5. **Learn patterns** - Advanced patterns and best practices
6. **Real-world use** - Apply to actual projects
7. **Performance** - Optimization and monitoring

## Additional Resources

- **Official Documentation**: https://kotlinlang.org/docs/coroutines-overview.html
- **Kotlin Coroutines GitHub**: https://github.com/Kotlin/kotlinx.coroutines
- **Tutorials and Guides**: Official Kotlin website
- **Sample Projects**: GitHub repositories with coroutine examples

## Version Compatibility

- **Kotlin 1.6+**: Full coroutines support
- **Kotlin 1.8+**: Enhanced Flow operators
- **Kotlin 1.9+**: Improved performance and stability
- **Coroutines 1.7+**: Latest features and improvements

## Troubleshooting

### Common Error Messages

**"Suspend function should be called only from a coroutine"**
- Solution: Call from inside `launch`, `async`, or another suspend function

**"CancellationException" during collection**
- Solution: Check for `isActive` or use cancelling operators correctly

**"Channel was closed"**
- Solution: Check if sender is still alive, handle channel closure properly

**"TimeoutCancellationException"**
- Solution: Increase timeout or check for hanging operations

---

**Congratulations!** You now have a complete reference for Kotlin coroutines. Keep practicing with real projects to master these concepts!