# Lesson 3: Exception Handling and Cancellation

## Understanding Coroutine Cancellation

Coroutines can be cancelled at any time. Proper cancellation handling is crucial for resource management and preventing memory leaks.

## 1. Basic Cancellation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("Job: Working on iteration $i")
            delay(500L)
        }
    }
    
    delay(1300L)  // Let it run for a bit
    println("Main: Cancelling the job...")
    job.cancel()  // Cancel the coroutine
    job.join()    // Wait for cancellation to complete
    println("Main: Job cancelled")
}

// Output:
// Job: Working on iteration 0
// Job: Working on iteration 1
// Job: Working on iteration 2
// Main: Cancelling the job...
// Main: Job cancelled
```

**Key Points:**
- `job.cancel()` requests cancellation
- `job.join()` waits for the coroutine to finish
- `job.cancelAndJoin()` combines both operations

## 2. Cancellation is Cooperative

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch(Dispatchers.Default) {
        var i = 0
        // This loop won't stop even after cancellation!
        while (i < 5) {
            Thread.sleep(500L)  // Not a suspend function
            println("Working: ${i++}")
        }
    }
    
    delay(1000L)
    println("Cancelling...")
    job.cancelAndJoin()
    println("Done")
}

// Output: All 5 iterations complete despite cancellation!
// Working: 0
// Working: 1
// Cancelling...
// Working: 2
// Working: 3
// Working: 4
// Done
```

**Problem:** The loop doesn't check for cancellation. Let's fix it:

## 3. Making Code Cancellable

**Method 1: Check isActive**

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch(Dispatchers.Default) {
        var i = 0
        while (isActive) {  // Check if coroutine is active
            Thread.sleep(500L)
            println("Working: ${i++}")
        }
    }
    
    delay(1300L)
    println("Cancelling...")
    job.cancelAndJoin()
    println("Done")
}

// Output:
// Working: 0
// Working: 1
// Working: 2
// Cancelling...
// Done
```

**Method 2: Use suspending functions**

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            delay(500L)  // Suspending function checks cancellation
            println("Working: $i")
        }
    }
    
    delay(1300L)
    println("Cancelling...")
    job.cancelAndJoin()
    println("Done")
}
```

**Method 3: Explicitly check and throw**

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch(Dispatchers.Default) {
        repeat(5) { i ->
            ensureActive()  // Throws CancellationException if cancelled
            Thread.sleep(500L)
            println("Working: $i")
        }
    }
    
    delay(1300L)
    println("Cancelling...")
    job.cancelAndJoin()
    println("Done")
}
```

## 4. Releasing Resources on Cancellation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("Processing item $i")
                delay(300L)
            }
        } finally {
            println("Cleaning up resources...")
            // Close files, connections, etc.
        }
    }
    
    delay(1000L)
    println("Cancelling job...")
    job.cancelAndJoin()
    println("Done")
}

// Output:
// Processing item 0
// Processing item 1
// Processing item 2
// Cancelling job...
// Cleaning up resources...
// Done
```

## 5. Non-Cancellable Block

Sometimes you need to perform cleanup that shouldn't be cancelled:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("Working: $i")
                delay(300L)
            }
        } finally {
            withContext(NonCancellable) {
                println("Closing resources (non-cancellable)...")
                delay(500L)  // This delay won't be cancelled
                println("Resources closed successfully")
            }
        }
    }
    
    delay(1000L)
    job.cancelAndJoin()
    println("Done")
}

// Output:
// Working: 0
// Working: 1
// Working: 2
// Closing resources (non-cancellable)...
// Resources closed successfully
// Done
```

## 6. Timeout Operations

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // withTimeout throws TimeoutCancellationException
    try {
        withTimeout(1300L) {
            repeat(1000) { i ->
                println("Item $i")
                delay(500L)
            }
        }
    } catch (e: TimeoutCancellationException) {
        println("Timed out!")
    }
    
    // withTimeoutOrNull returns null on timeout
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("Processing $i")
            delay(500L)
        }
        "Completed"
    }
    
    println("Result: $result")
}

// Output:
// Item 0
// Item 1
// Item 2
// Timed out!
// Processing 0
// Processing 1
// Processing 2
// Result: null
```

## 7. Exception Handling in Coroutines

**Launch vs Async Exception Behavior**

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // launch: Exception is thrown immediately
    val job = launch {
        throw RuntimeException("Exception in launch!")
    }
    
    delay(100L)
    println("After launch exception")
    
    // async: Exception is thrown when you call await()
    val deferred = async {
        throw RuntimeException("Exception in async!")
    }
    
    delay(100L)
    println("After async creation")
    
    try {
        deferred.await()
    } catch (e: RuntimeException) {
        println("Caught: ${e.message}")
    }
}

// Output:
// Exception in thread "main" RuntimeException: Exception in launch!
// (Program crashes before other output)
```

## 8. CoroutineExceptionHandler

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught exception: ${exception.message}")
    }
    
    val job = GlobalScope.launch(handler) {
        throw RuntimeException("Something went wrong!")
    }
    
    job.join()
    println("Program continues...")
}

// Output:
// Caught exception: Something went wrong!
// Program continues...
```

**Important:** CoroutineExceptionHandler only works with `launch`, not `async`.

## 9. Supervised Job - Independent Children

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    
    with(CoroutineScope(coroutineContext + supervisor)) {
        // Child 1 - will fail
        val child1 = launch {
            println("Child 1: Starting")
            delay(500L)
            throw RuntimeException("Child 1 failed!")
        }
        
        // Child 2 - will succeed
        val child2 = launch {
            println("Child 2: Starting")
            repeat(3) { i ->
                delay(400L)
                println("Child 2: $i")
            }
            println("Child 2: Completed")
        }
        
        joinAll(child1, child2)
    }
    
    println("Both children finished")
}

// Output:
// Child 1: Starting
// Child 2: Starting
// Child 2: 0
// Exception in thread "DefaultDispatcher-worker-1" RuntimeException: Child 1 failed!
// Child 2: 1
// Child 2: 2
// Child 2: Completed
// Both children finished
```

## 10. supervisorScope - Failure Isolation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Handler caught: ${exception.message}")
    }
    
    supervisorScope {
        val child1 = launch(handler) {
            println("Child 1: Working")
            delay(500L)
            throw RuntimeException("Child 1 crashed")
        }
        
        val child2 = launch {
            println("Child 2: Working")
            repeat(5) { i ->
                delay(300L)
                println("Child 2: Step $i")
            }
        }
        
        joinAll(child1, child2)
    }
    
    println("Supervisor scope completed")
}

// Output:
// Child 1: Working
// Child 2: Working
// Child 2: Step 0
// Handler caught: Child 1 crashed
// Child 2: Step 1
// Child 2: Step 2
// Child 2: Step 3
// Child 2: Step 4
// Supervisor scope completed
```

## 11. Practical Example: Robust API Client

```kotlin
import kotlinx.coroutines.*

class ApiClient {
    private val handler = CoroutineExceptionHandler { _, exception ->
        println("API Error: ${exception.message}")
    }
    
    suspend fun fetchDataWithTimeout(url: String): String? {
        return withTimeoutOrNull(3000L) {
            try {
                fetchData(url)
            } catch (e: Exception) {
                println("Error fetching $url: ${e.message}")
                null
            }
        }
    }
    
    private suspend fun fetchData(url: String): String {
        delay(2000L)  // Simulate network delay
        
        // Simulate random failures
        if (url.contains("fail")) {
            throw RuntimeException("Network error")
        }
        
        return "Data from $url"
    }
}

fun main() = runBlocking {
    val client = ApiClient()
    
    supervisorScope {
        val results = listOf(
            async { client.fetchDataWithTimeout("api/users") },
            async { client.fetchDataWithTimeout("api/fail/posts") },
            async { client.fetchDataWithTimeout("api/comments") }
        )
        
        results.awaitAll().forEachIndexed { index, result ->
            println("Result $index: $result")
        }
    }
    
    println("All requests completed")
}

// Output:
// Error fetching api/fail/posts: Network error
// Result 0: Data from api/users
// Result 1: null
// Result 2: Data from api/comments
// All requests completed
```

## Key Takeaways from Lesson 3

1. **Cancellation is cooperative** - Code must check `isActive` or use suspending functions
2. **Use try-finally** - For resource cleanup
3. **NonCancellable** - For critical cleanup that shouldn't be cancelled
4. **withTimeout/withTimeoutOrNull** - Automatic timeout handling
5. **CoroutineExceptionHandler** - Centralized exception handling
6. **SupervisorJob/supervisorScope** - Children fail independently
7. **launch** - Propagates exceptions immediately
8. **async** - Exceptions thrown on `await()`

## Practice Exercise

Create a data processing pipeline that:
1. Fetches 3 resources concurrently
2. One resource should fail
3. Other resources should continue processing
4. Handle errors gracefully
5. Use timeout of 2 seconds
6. Clean up resources properly

<details>
<summary>Solution</summary>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, e ->
        println("Error handled: ${e.message}")
    }
    
    supervisorScope {
        val jobs = listOf(
            async(handler) {
                withTimeoutOrNull(2000L) {
                    try {
                        fetchResource("Resource 1", 500L, false)
                    } finally {
                        println("Cleanup Resource 1")
                    }
                }
            },
            async(handler) {
                withTimeoutOrNull(2000L) {
                    try {
                        fetchResource("Resource 2", 1000L, true)
                    } finally {
                        println("Cleanup Resource 2")
                    }
                }
            },
            async(handler) {
                withTimeoutOrNull(2000L) {
                    try {
                        fetchResource("Resource 3", 800L, false)
                    } finally {
                        println("Cleanup Resource 3")
                    }
                }
            }
        )
        
        val results = jobs.awaitAll()
        println("Results: $results")
    }
}

suspend fun fetchResource(name: String, delayMs: Long, shouldFail: Boolean): String {
    delay(delayMs)
    if (shouldFail) throw RuntimeException("$name failed")
    return "$name data"
}

// Output:
// Cleanup Resource 1
// Error handled: Resource 2 failed
// Cleanup Resource 2
// Cleanup Resource 3
// Results: [Resource 1 data, null, Resource 3 data]
```
</details>

---

Next: [Lesson 4: Flow - Asynchronous Data Streams](lesson-04-flow-streams.md)