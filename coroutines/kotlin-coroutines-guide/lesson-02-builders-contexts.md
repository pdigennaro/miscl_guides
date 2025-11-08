# Lesson 2: Coroutine Builders and Contexts

## Coroutine Builders Overview

Coroutine builders are functions that create and start coroutines. The three main builders are:
- **`launch`** - Fire and forget (returns Job)
- **`async`** - Returns a result (returns Deferred)
- **`runBlocking`** - Blocks the thread (mainly for main functions and tests)

## 1. Launch Builder - Fire and Forget

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        delay(1000L)
        println("Launch: Task completed!")
    }
    
    println("Launch: Started coroutine")
    job.join()  // Wait for the coroutine to complete
    println("Launch: All done")
}

// Output:
// Launch: Started coroutine
// Launch: Task completed!
// Launch: All done
```

**Key Points:**
- `launch` returns a `Job` object
- Use `Job.join()` to wait for completion
- Doesn't return a result value
- Perfect for tasks that don't need to return data

## 2. Async Builder - Concurrent Computation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val deferred = async {
        delay(1000L)
        "Result from async"
    }
    
    println("Async: Started coroutine")
    val result = deferred.await()  // Wait and get the result
    println("Async: $result")
}

// Output:
// Async: Started coroutine
// Async: Result from async
```

**Key Points:**
- `async` returns a `Deferred<T>` object
- Use `await()` to get the result
- Perfect for concurrent computations that return values

## Comparing Launch vs Async

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    // Sequential execution with launch
    val timeSequential = measureTimeMillis {
        val job1 = launch { task1() }
        val job2 = launch { task2() }
        job1.join()
        job2.join()
    }
    println("Sequential with launch: $timeSequential ms")
    
    // Concurrent execution with async
    val timeConcurrent = measureTimeMillis {
        val result1 = async { computeTask1() }
        val result2 = async { computeTask2() }
        println("Sum: ${result1.await() + result2.await()}")
    }
    println("Concurrent with async: $timeConcurrent ms")
}

suspend fun task1() {
    delay(1000L)
    println("Task 1 done")
}

suspend fun task2() {
    delay(1000L)
    println("Task 2 done")
}

suspend fun computeTask1(): Int {
    delay(1000L)
    return 10
}

suspend fun computeTask2(): Int {
    delay(1000L)
    return 20
}

// Output:
// Task 1 done
// Task 2 done
// Sequential with launch: ~1000 ms
// Sum: 30
// Concurrent with async: ~1000 ms
```

## 3. Coroutine Context and Dispatchers

Every coroutine runs in a **CoroutineContext** which includes:
- **Dispatcher** - Determines which thread(s) the coroutine uses
- **Job** - Controls the lifecycle
- **CoroutineName** - Name for debugging
- **Exception Handler** - Handles uncaught exceptions

## Dispatchers Explained

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Dispatchers.Default - CPU-intensive work
    launch(Dispatchers.Default) {
        println("Default: ${Thread.currentThread().name}")
        // Good for: computational tasks, sorting, parsing
    }
    
    // Dispatchers.IO - I/O operations
    launch(Dispatchers.IO) {
        println("IO: ${Thread.currentThread().name}")
        // Good for: network requests, file operations, database queries
    }
    
    // Dispatchers.Main - UI thread (Android/JavaFX)
    // launch(Dispatchers.Main) {
    //     // Update UI elements
    // }
    
    // Dispatchers.Unconfined - Inherits context
    launch(Dispatchers.Unconfined) {
        println("Unconfined 1: ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined 2: ${Thread.currentThread().name}")
    }
    
    delay(1000L)
}

// Sample Output:
// Default: DefaultDispatcher-worker-1
// IO: DefaultDispatcher-worker-2
// Unconfined 1: main
// Unconfined 2: kotlinx.coroutines.DefaultExecutor
```

## Switching Contexts with withContext

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Main: ${Thread.currentThread().name}")
    
    val result = withContext(Dispatchers.Default) {
        println("withContext: ${Thread.currentThread().name}")
        performHeavyComputation()
    }
    
    println("Result: $result on ${Thread.currentThread().name}")
}

suspend fun performHeavyComputation(): Int {
    delay(1000L)  // Simulate heavy work
    return 42
}

// Output:
// Main: main
// withContext: DefaultDispatcher-worker-1
// Result: 42 on main
```

**Key Point:** `withContext` switches context, executes code, and returns to the original context.

## 4. Job Hierarchy and Cancellation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val parentJob = launch {
        val child1 = launch {
            repeat(5) { i ->
                delay(500L)
                println("Child 1: $i")
            }
        }
        
        val child2 = launch {
            repeat(5) { i ->
                delay(300L)
                println("Child 2: $i")
            }
        }
    }
    
    delay(1200L)
    println("Cancelling parent job...")
    parentJob.cancel()  // Cancels all children too
    parentJob.join()
    println("Parent job cancelled")
}

// Output:
// Child 2: 0
// Child 1: 0
// Child 2: 1
// Child 1: 1
// Child 2: 2
// Cancelling parent job...
// Parent job cancelled
```

## Practical Example: Parallel Data Fetching

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

data class User(val id: Int, val name: String)
data class Posts(val userId: Int, val posts: List<String>)
data class Profile(val userId: Int, val bio: String)

fun main() = runBlocking {
    val time = measureTimeMillis {
        val userId = 1
        
        // Fetch all data concurrently
        val user = async(Dispatchers.IO) { fetchUser(userId) }
        val posts = async(Dispatchers.IO) { fetchPosts(userId) }
        val profile = async(Dispatchers.IO) { fetchProfile(userId) }
        
        // Wait for all results
        displayUserData(user.await(), posts.await(), profile.await())
    }
    
    println("Total time: $time ms")
}

suspend fun fetchUser(id: Int): User {
    delay(1000L)  // Simulate API call
    return User(id, "John Doe")
}

suspend fun fetchPosts(userId: Int): Posts {
    delay(800L)  // Simulate API call
    return Posts(userId, listOf("Post 1", "Post 2", "Post 3"))
}

suspend fun fetchProfile(userId: Int): Profile {
    delay(600L)  // Simulate API call
    return Profile(userId, "Software Developer")
}

fun displayUserData(user: User, posts: Posts, profile: Profile) {
    println("=== User Dashboard ===")
    println("Name: ${user.name}")
    println("Bio: ${profile.bio}")
    println("Posts: ${posts.posts.joinToString(", ")}")
}

// Output:
// === User Dashboard ===
// Name: John Doe
// Bio: Software Developer
// Posts: Post 1, Post 2, Post 3
// Total time: ~1000 ms (not 2400 ms!)
```

## Coroutine Scope Best Practices

```kotlin
import kotlinx.coroutines.*

class DataRepository {
    // Create a custom scope for this class
    private val scope = CoroutineScope(Dispatchers.Default + Job())
    
    fun fetchData() {
        scope.launch {
            val data = withContext(Dispatchers.IO) {
                // Fetch from network
                delay(1000L)
                "Data from server"
            }
            println("Fetched: $data")
        }
    }
    
    fun cleanup() {
        scope.cancel()  // Cancel all coroutines when done
    }
}

fun main() = runBlocking {
    val repository = DataRepository()
    repository.fetchData()
    delay(1500L)
    repository.cleanup()
}
```

## Key Takeaways from Lesson 2

1. **`launch`** - Use for fire-and-forget tasks (returns Job)
2. **`async`** - Use when you need a result (returns Deferred)
3. **Dispatchers.Default** - CPU-intensive work
4. **Dispatchers.IO** - I/O operations (network, files, database)
5. **`withContext`** - Switch context and return to original
6. **Job hierarchy** - Cancelling parent cancels all children
7. **Custom scopes** - Use for managing coroutines in classes

## Practice Exercise

Create a program that:
1. Fetches 3 different resources concurrently using `async`
2. Each fetch takes different time (500ms, 1000ms, 1500ms)
3. Combine the results and print them
4. Measure total execution time

<details>
<summary>Solution</summary>

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val time = measureTimeMillis {
        val resource1 = async { fetchResource1() }
        val resource2 = async { fetchResource2() }
        val resource3 = async { fetchResource3() }
        
        val combined = "${resource1.await()}, ${resource2.await()}, ${resource3.await()}"
        println("Combined: $combined")
    }
    println("Execution time: $time ms")
}

suspend fun fetchResource1(): String {
    delay(500L)
    return "Resource 1"
}

suspend fun fetchResource2(): String {
    delay(1000L)
    return "Resource 2"
}

suspend fun fetchResource3(): String {
    delay(1500L)
    return "Resource 3"
}

// Output:
// Combined: Resource 1, Resource 2, Resource 3
// Execution time: ~1500 ms
```
</details>

---

Next: [Lesson 3: Exception Handling and Cancellation](lesson-03-exception-cancellation.md)