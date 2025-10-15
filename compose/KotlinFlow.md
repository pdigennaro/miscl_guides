# Kotlin Flow: Complete Guide

Kotlin Flow is a cold asynchronous stream that sequentially emits values and completes normally or with an exception. It's part of Kotlin Coroutines and is built on top of coroutines.

## 1. Basic Flow Creation

### Simple Flow
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun simpleFlow(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // Simulate async work
        emit(i) // Emit values
    }
}

suspend fun main() {
    simpleFlow().collect { value ->
        println("Received: $value")
    }
}
```
**Output:**
```
Received: 1
Received: 2
Received: 3
```

### Flow Builders

```kotlin
suspend fun main() {
    // flowOf - creates flow from fixed values
    flowOf(1, 2, 3, 4, 5)
        .collect { println("flowOf: $it") }
    
    // asFlow - converts collections to flow
    listOf("A", "B", "C").asFlow()
        .collect { println("asFlow: $it") }
}
```
**Output:**
```
flowOf: 1
flowOf: 2
flowOf: 3
flowOf: 4
flowOf: 5
asFlow: A
asFlow: B
asFlow: C
```

## 2. Flow is Cold

Flows are cold streams - they don't produce values until collected.

```kotlin
suspend fun main() {
    val flow = flow {
        println("Flow started")
        for (i in 1..3) {
            emit(i)
        }
    }
    
    println("Before collection")
    flow.collect { println("Collected: $it") }
    println("After collection")
    
    println("\nCollecting again:")
    flow.collect { println("Collected: $it") }
}
```
**Output:**
```
Before collection
Flow started
Collected: 1
Collected: 2
Collected: 3
After collection

Collecting again:
Flow started
Collected: 1
Collected: 2
Collected: 3
```

## 3. Flow Cancellation

Flows are automatically cancelled when the coroutine is cancelled.

```kotlin
suspend fun main() {
    withTimeoutOrNull(250) {
        flow {
            for (i in 1..5) {
                delay(100)
                emit(i)
                println("Emitted: $i")
            }
        }.collect { value ->
            println("Collected: $value")
        }
    }
    println("Done")
}
```
**Output:**
```
Emitted: 1
Collected: 1
Emitted: 2
Collected: 2
Done
```

## 4. Flow Operators

### Transform Operators

```kotlin
suspend fun main() {
    // map
    (1..3).asFlow()
        .map { it * it }
        .collect { println("map: $it") }
    
    println()
    
    // filter
    (1..5).asFlow()
        .filter { it % 2 == 0 }
        .collect { println("filter: $it") }
    
    println()
    
    // transform - can emit multiple values
    (1..3).asFlow()
        .transform { value ->
            emit("String: $value")
            emit(value * 2)
        }
        .collect { println("transform: $it") }
}
```
**Output:**
```
map: 1
map: 4
map: 9

filter: 2
filter: 4

transform: String: 1
transform: 2
transform: String: 2
transform: 4
transform: String: 3
transform: 6
```

### Take Operators

```kotlin
suspend fun main() {
    val numbers = (1..10).asFlow()
    
    println("take(3):")
    numbers.take(3).collect { println(it) }
    
    println("\ntakeWhile:")
    numbers.takeWhile { it < 5 }.collect { println(it) }
}
```
**Output:**
```
take(3):
1
2
3

takeWhile:
1
2
3
4
```

## 5. Terminal Operators

Terminal operators are suspending functions that start collection.

```kotlin
suspend fun main() {
    val flow = (1..5).asFlow()
    
    // collect - basic terminal operator
    println("collect:")
    flow.collect { print("$it ") }
    
    // toList
    println("\n\ntoList: ${flow.toList()}")
    
    // toSet
    println("toSet: ${flow.toSet()}")
    
    // first
    println("first: ${flow.first()}")
    
    // last
    println("last: ${flow.last()}")
    
    // reduce
    val sum = flow.reduce { acc, value -> acc + value }
    println("reduce (sum): $sum")
    
    // fold
    val product = flow.fold(1) { acc, value -> acc * value }
    println("fold (product): $product")
}
```
**Output:**
```
collect:
1 2 3 4 5 

toList: [1, 2, 3, 4, 5]
toSet: [1, 2, 3, 4, 5]
first: 1
last: 5
reduce (sum): 15
fold (product): 120
```

## 6. Flow Context

Flows preserve context and don't switch coroutine contexts.

```kotlin
fun simpleFlow(): Flow<Int> = flow {
    println("Flow started in: ${Thread.currentThread().name}")
    for (i in 1..3) {
        emit(i)
    }
}

suspend fun main() {
    simpleFlow()
        .collect { value ->
            println("Collected $value in: ${Thread.currentThread().name}")
        }
}
```
**Output:**
```
Flow started in: main
Collected 1 in: main
Collected 2 in: main
Collected 3 in: main
```

### flowOn Operator

Changes the context of the flow emission.

```kotlin
suspend fun main() = coroutineScope {
    flow {
        println("Emitting in: ${Thread.currentThread().name}")
        emit(1)
        emit(2)
    }
    .flowOn(Dispatchers.Default)
    .collect { value ->
        println("Collected $value in: ${Thread.currentThread().name}")
    }
}
```
**Output:**
```
Emitting in: DefaultDispatcher-worker-1
Collected 1 in: main
Collected 2 in: main
```

## 7. Buffering

### buffer

Allows emission and collection to run concurrently.

```kotlin
suspend fun main() {
    val time = measureTimeMillis {
        flow {
            for (i in 1..3) {
                delay(100) // Emission takes time
                emit(i)
            }
        }
        .buffer() // Buffer emissions
        .collect { value ->
            delay(300) // Processing takes time
            println("Processed: $value")
        }
    }
    println("Total time: $time ms")
}
```
**Output:**
```
Processed: 1
Processed: 2
Processed: 3
Total time: ~1000 ms (without buffer: ~1200 ms)
```

### conflate

Skips intermediate values when collector is slow.

```kotlin
suspend fun main() {
    flow {
        for (i in 1..5) {
            delay(100)
            emit(i)
            println("Emitted: $i")
        }
    }
    .conflate()
    .collect { value ->
        delay(300)
        println("Collected: $value")
    }
}
```
**Output:**
```
Emitted: 1
Collected: 1
Emitted: 2
Emitted: 3
Emitted: 4
Collected: 4
Emitted: 5
Collected: 5
```

### collectLatest

Cancels previous collection when new value arrives.

```kotlin
suspend fun main() {
    flow {
        for (i in 1..5) {
            delay(100)
            emit(i)
        }
    }
    .collectLatest { value ->
        println("Collecting $value")
        delay(300)
        println("Completed $value")
    }
}
```
**Output:**
```
Collecting 1
Collecting 2
Collecting 3
Collecting 4
Collecting 5
Completed 5
```

## 8. Combining Flows

### zip

Combines two flows by pairing corresponding values.

```kotlin
suspend fun main() {
    val numbers = (1..3).asFlow()
    val strings = flowOf("one", "two", "three")
    
    numbers.zip(strings) { num, str -> "$num -> $str" }
        .collect { println(it) }
}
```
**Output:**
```
1 -> one
2 -> two
3 -> three
```

### combine

Combines flows, emitting whenever any flow emits.

```kotlin
suspend fun main() {
    val numbers = flow {
        emit(1)
        delay(150)
        emit(2)
        delay(150)
        emit(3)
    }
    
    val strings = flow {
        emit("A")
        delay(200)
        emit("B")
        delay(200)
        emit("C")
    }
    
    numbers.combine(strings) { num, str -> "$num$str" }
        .collect { println(it) }
}
```
**Output:**
```
1A
2A
2B
3B
3C
```

## 9. Flattening Flows

### flatMapConcat

Processes flows sequentially.

```kotlin
suspend fun main() {
    (1..3).asFlow()
        .flatMapConcat { value ->
            flow {
                emit("$value: First")
                delay(100)
                emit("$value: Second")
            }
        }
        .collect { println(it) }
}
```
**Output:**
```
1: First
1: Second
2: First
2: Second
3: First
3: Second
```

### flatMapMerge

Processes flows concurrently.

```kotlin
suspend fun main() {
    (1..3).asFlow()
        .flatMapMerge { value ->
            flow {
                emit("$value: First")
                delay(100)
                emit("$value: Second")
            }
        }
        .collect { println(it) }
}
```
**Output:**
```
1: First
2: First
3: First
1: Second
2: Second
3: Second
```

### flatMapLatest

Cancels previous flow when new value arrives.

```kotlin
suspend fun main() {
    (1..3).asFlow()
        .onEach { delay(100) }
        .flatMapLatest { value ->
            flow {
                emit("$value: First")
                delay(150)
                emit("$value: Second")
            }
        }
        .collect { println(it) }
}
```
**Output:**
```
1: First
2: First
3: First
3: Second
```

## 10. Exception Handling

### try-catch

```kotlin
suspend fun main() {
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Error!")
        emit(3)
    }
    .catch { e -> println("Caught: ${e.message}") }
    .collect { println("Collected: $it") }
}
```
**Output:**
```
Collected: 1
Collected: 2
Caught: Error!
```

### onCompletion

Called when flow completes (successfully or with exception).

```kotlin
suspend fun main() {
    flow {
        emit(1)
        emit(2)
    }
    .onCompletion { cause ->
        if (cause != null) println("Flow failed: $cause")
        else println("Flow completed successfully")
    }
    .collect { println("Collected: $it") }
}
```
**Output:**
```
Collected: 1
Collected: 2
Flow completed successfully
```

## 11. StateFlow and SharedFlow

### StateFlow

Hot flow that always has a value and emits updates.

```kotlin
class CounterViewModel {
    private val _counter = MutableStateFlow(0)
    val counter: StateFlow<Int> = _counter
    
    fun increment() {
        _counter.value++
    }
}

suspend fun main() = coroutineScope {
    val viewModel = CounterViewModel()
    
    launch {
        viewModel.counter.collect { value ->
            println("Observer 1: $value")
        }
    }
    
    delay(100)
    viewModel.increment()
    delay(100)
    viewModel.increment()
    delay(100)
}
```
**Output:**
```
Observer 1: 0
Observer 1: 1
Observer 1: 2
```

### SharedFlow

Hot flow that can have multiple subscribers.

```kotlin
suspend fun main() = coroutineScope {
    val sharedFlow = MutableSharedFlow<Int>()
    
    launch {
        sharedFlow.collect { println("Collector 1: $it") }
    }
    
    launch {
        sharedFlow.collect { println("Collector 2: $it") }
    }
    
    delay(100)
    sharedFlow.emit(1)
    delay(100)
    sharedFlow.emit(2)
    delay(100)
}
```
**Output:**
```
Collector 1: 1
Collector 2: 1
Collector 1: 2
Collector 2: 2
```

## 12. Practical Example: Network Request with Retry

```kotlin
suspend fun main() {
    var attemptCount = 0
    
    flow {
        attemptCount++
        println("Attempt $attemptCount")
        if (attemptCount < 3) {
            throw IOException("Network error")
        }
        emit("Success!")
    }
    .retry(3) { e ->
        println("Retrying due to: ${e.message}")
        delay(1000)
        true
    }
    .catch { e -> emit("Failed after retries: ${e.message}") }
    .collect { println("Result: $it") }
}
```
**Output:**
```
Attempt 1
Retrying due to: Network error
Attempt 2
Retrying due to: Network error
Attempt 3
Result: Success!
```

This guide covers the essential aspects of Kotlin Flow. Flows are powerful for handling asynchronous data streams in a declarative and efficient way!