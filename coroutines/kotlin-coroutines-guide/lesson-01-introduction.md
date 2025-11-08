# Lesson 1: Introduction to Coroutines

## What Are Coroutines?

Coroutines are Kotlin's approach to asynchronous programming. They allow you to write non-blocking, concurrent code that looks and behaves like sequential code. Think of coroutines as lightweight threads that can be suspended and resumed without blocking the underlying thread.

### Key Benefits
- **Lightweight**: You can run thousands of coroutines without significant memory overhead
- **Readable**: Asynchronous code looks like synchronous code
- **Structured Concurrency**: Built-in mechanisms to prevent memory leaks and ensure proper cleanup
- **Non-blocking**: Efficient resource utilization

## Coroutines vs Threads

```kotlin
// Traditional Thread (Heavy, blocks the thread)
thread {
    Thread.sleep(1000)
    println("World!")
}
println("Hello,")

// Coroutine (Lightweight, suspends without blocking)
GlobalScope.launch {
    delay(1000)  // Suspends, doesn't block
    println("World!")
}
println("Hello,")
```

## Setting Up Dependencies

Add these to your `build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

## Your First Coroutine

```kotlin
import kotlinx.coroutines.*

fun main() {
    // Launch a coroutine
    GlobalScope.launch {
        delay(1000L)  // Non-blocking delay
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)  // Keep JVM alive
}

// Output:
// Hello,
// World!
```

**Explanation:**
- `GlobalScope.launch` creates a new coroutine
- `delay()` suspends the coroutine for 1 second without blocking the thread
- The main thread continues and prints "Hello," immediately
- After 1 second, the coroutine resumes and prints "World!"

## Blocking vs Suspending

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  // Creates a coroutine that blocks the main thread
    launch {  // Launches a new coroutine
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}

// Output:
// Hello,
// World!
```

**Key Concepts:**
- `runBlocking`: Bridges blocking and non-blocking worlds; blocks the current thread
- `launch`: Launches a coroutine without blocking the current thread
- `delay()`: A suspending function (only works inside coroutines)

## Structured Concurrency

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope {  // Creates a coroutine scope
        launch {
            delay(500L)
            println("Task from nested launch")
        }
        
        delay(100L)
        println("Task from coroutine scope")
    }
    
    println("Coroutine scope is over")
}

// Output:
// Task from coroutine scope
// Task from runBlocking
// Task from nested launch
// Coroutine scope is over
```

**Important:** `coroutineScope` waits for all its children to complete before returning.

## Practical Example: Multiple Concurrent Tasks

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)  // Pretend we're doing something useful
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)  // Pretend we're doing something useful
    return 29
}

// Output:
// The answer is 42
// Completed in approximately 1000 ms (not 2000!)
```

## Key Takeaways from Lesson 1

1. **Coroutines are lightweight** - You can create thousands without performance issues
2. **Suspend, don't block** - Use `delay()` instead of `Thread.sleep()`
3. **Structured concurrency** - Parent coroutines wait for children to complete
4. **runBlocking** - Use for bridging regular and coroutine code (mainly in main functions and tests)
5. **launch** - Fire and forget coroutine builder
6. **async/await** - For concurrent execution with results

## Practice Exercise

Try creating a coroutine that:
1. Prints "Starting..."
2. Waits 2 seconds
3. Prints "Middle..."
4. Waits 1 second
5. Prints "Done!"

<details>
<summary>Solution</summary>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("Starting...")
        delay(2000L)
        println("Middle...")
        delay(1000L)
        println("Done!")
    }
}
```
</details>

---

Next: [Lesson 2: Coroutine Builders and Contexts](lesson-02-builders-contexts.md)