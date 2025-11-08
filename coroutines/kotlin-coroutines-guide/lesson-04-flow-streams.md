# Lesson 4: Flow - Asynchronous Data Streams

## What is Flow?

Flow is Kotlin's solution for representing asynchronous streams of data. It's like a sequence, but each value is produced asynchronously, and the consumer can process values as they arrive.

### Key Characteristics
- **Cold stream** - Values are only produced when collected
- **Asynchronous** - Each value can be produced on a different thread
- **Cancelable** - Flow collection can be cancelled
- **Reactive** - Can process and transform data streams

## 1. Creating Flows

**Basic Flow Creation**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3, 4, 5).collect { value ->
        println("Received: $value")
    }
    
    listOf(1, 2, 3).asFlow().collect { value ->
        println("From list: $value")
    }
}

// Output:
// Received: 1
// Received: 2
// Received: 3
// Received: 4
// Received: 5
// From list: 1
// From list: 2
// From list: 3
```

**Flow Builder - flow {}**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow = flow {
        println("Flow started")
        for (i in 1..3) {
            delay(500L)  // Simulate async work
            emit(i)  // Emit value
        }
        println("Flow completed")
    }
    
    flow.collect { value ->
        println("Collected: $value")
    }
}

// Output:
// Flow started
// Collected: 1
// Collected: 2
// Collected: 3
// Flow completed
```

## 2. Flow is Cold

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow = flow {
        println("Flow is cold - only runs when collected")
        emit(1)
        emit(2)
        emit(3)
    }
    
    println("Flow created but not collected")
    delay(1000L)
    println("Now collecting...")
    flow.collect { value ->
        println("Value: $value")
    }
}

// Output:
// Flow created but not collected
// Now collecting...
// Flow is cold - only runs when collected
// Value: 1
// Value: 2
// Value: 3
```

## 3. Terminal Operations

**collect() - Consume all values**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3, 4, 5).collect { value ->
        println("Value: $value")
    }
}

// Output:
// Value: 1
// Value: 2
// Value: 3
// Value: 4
// Value: 5
```

**toList() / toSet() - Collect to collection**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val list = flowOf("A", "B", "C").toList()
    println("List: $list")
    
    val set = flowOf(1, 2, 2, 3, 3, 3).toSet()
    println("Set: $set")
    
    val first = flowOf(10, 20, 30).first()
    println("First: $first")
    
    val count = flowOf(1, 2, 3, 4, 5).count()
    println("Count: $count")
}

// Output:
// List: [A, B, C]
// Set: [1, 2, 3]
// First: 10
// Count: 5
```

## 4. Intermediate Operators

**map() - Transform values**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3, 4, 5)
        .map { it * it }  // Square each value
        .collect { value ->
            println("Square: $value")
        }
}

// Output:
// Square: 1
// Square: 4
// Square: 9
// Square: 16
// Square: 25
```

**filter() - Filter values**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .filter { it % 2 == 0 }  // Only even numbers
        .collect { value ->
            println("Even: $value")
        }
}

// Output:
// Even: 2
// Even: 4
// Even: 6
// Even: 8
// Even: 10
```

**take() - Limit values**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .take(3)  // Take only first 3 values
        .collect { value ->
            println("Taken: $value")
        }
}

// Output:
// Taken: 1
// Taken: 2
// Taken: 3
```

## 5. Combining Flows

**zip() - Combine two flows**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow1 = flowOf(1, 2, 3)
    val flow2 = flowOf("A", "B", "C")
    
    flow1.zip(flow2) { num, letter -> "$num$letter" }
        .collect { pair ->
            println("Zipped: $pair")
        }
}

// Output:
// Zipped: 1A
// Zipped: 2B
// Zipped: 3C
```

**combine() - Combine latest values**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow1 = (1..3).asFlow().onEach { delay(100L) }
    val flow2 = flowOf("X", "Y", "Z").onEach { delay(200L) }
    
    flow1.combine(flow2) { num, letter -> "$num$letter" }
        .collect { pair ->
            println("Combined: $pair")
        }
}

// Output:
// Combined: 3X
// Combined: 3Y
// Combined: 3Z
```

## 6. Chaining Operations

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    (1..10).asFlow()
        .filter { it % 2 == 0 }           // Keep even numbers
        .map { it * it }                   // Square them
        .take(3)                           // Take first 3
        .collect { result ->
            println("Result: $result")
        }
}

// Output:
// Result: 4
// Result: 16
// Result: 36
```

## 7. Flattening Operations

**flattenConcat() - Concatenate multiple flows**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flowOfFlows = flowOf(
        flowOf(1, 2, 3),
        flowOf(4, 5, 6),
        flowOf(7, 8, 9)
    )
    
    flowOfFlows.flattenConcat()
        .collect { value ->
            println("Flattened: $value")
        }
}

// Output:
// Flattened: 1
// Flattened: 2
// Flattened: 3
// Flattened: 4
// Flattened: 5
// Flattened: 6
// Flattened: 7
// Flattened: 8
// Flattened: 9
```

**flatMapConcat() - Transform and concatenate**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3)
        .flatMapConcat { number ->
            flow {
                emit("Group: $number")
                for (i in 1..3) {
                    delay(200L)
                    emit("$number.$i")
                }
            }
        }
        .collect { value ->
            println("Flat: $value")
        }
}

// Output:
// Flat: Group: 1
// Flat: 1.1
// Flat: 1.2
// Flat: 1.3
// Flat: Group: 2
// Flat: 2.1
// Flat: 2.2
// Flat: 2.3
// Flat: Group: 3
// Flat: 3.1
// Flat: 3.2
// Flat: 3.3
```

**flatMapMerge() - Transform and merge concurrently**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flowOf(1, 2, 3)
        .flatMapMerge(2) { number ->
            flow {
                for (i in 1..3) {
                    delay(200L)
                    emit("$number.$i")
                }
            }
        }
        .collect { value ->
            println("Merged: $value")
        }
}

// Output (order may vary due to concurrency):
// Merged: 1.1
// Merged: 2.1
// Merged: 1.2
// Merged: 3.1
// Merged: 1.3
// Merged: 2.2
// Merged: 2.3
// Merged: 3.2
// Merged: 3.3
```

## 8. Error Handling in Flow

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow = flow {
        emit(1)
        emit(2)
        emit(3)
        throw RuntimeException("Something went wrong!")
        emit(4)  // This won't be emitted
    }
    
    try {
        flow.collect { value ->
            println("Value: $value")
        }
    } catch (e: Exception) {
        println("Error caught: ${e.message}")
    }
}

// Output:
// Value: 1
// Value: 2
// Value: 3
// Error caught: Something went wrong!
```

**catch() - Handle errors**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow = flow {
        emit(1)
        emit(2)
        emit(3)
        throw RuntimeException("Network error!")
        emit(4)
    }
    
    flow.catch { exception ->
        println("Caught error: ${exception.message}")
        emit(-1)  // Emit default value on error
    }
    .collect { value ->
        println("Collected: $value")
    }
}

// Output:
// Collected: 1
// Collected: 2
// Collected: 3
// Caught error: Network error!
// Collected: -1
```

## 9. Flow Context Preservation

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flow {
        println("Flow context: ${Thread.currentThread().name}")
        emit("Data")
    }
    .map { value ->
        println("Map context: ${Thread.currentThread().name}")
        value.uppercase()
    }
    .collect { value ->
        println("Collect context: ${Thread.currentThread().name}")
        println("Collected: $value")
    }
}

// Output:
// Flow context: main
// Map context: main
// Collect context: main
// Collected: DATA
```

**withContext() - Switch context for emission**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flow {
        emit(1)
    }
    .map { value ->
        withContext(Dispatchers.Default) {
            println("Processing on: ${Thread.currentThread().name}")
            value * 2
        }
    }
    .collect { value ->
        println("Result: $value")
    }
}

// Output:
// Processing on: DefaultDispatcher-worker-1
// Result: 2
```

## 10. State Flow and Shared Flow

**StateFlow - Holds current value**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import java.util.*

class Counter {
    private val _state = MutableStateFlow(0)
    val state: StateFlow<Int> = _state
    
    fun increment() {
        _state.value++
    }
}

fun main() = runBlocking {
    val counter = Counter()
    
    // Collect state changes
    counter.state.collect { value ->
        println("Current state: $value")
    }
    
    delay(100L)
    counter.increment()
    counter.increment()
    counter.increment()
    delay(100L)
}

// Output:
// Current state: 0
// Current state: 1
// Current state: 2
// Current state: 3
```

**SharedFlow - Emits to multiple collectors**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class EventBus {
    private val _events = MutableSharedFlow<String>()
    val events: SharedFlow<String> = _events
    
    fun sendEvent(event: String) {
        _events.tryEmit(event)
    }
}

fun main() = runBlocking {
    val eventBus = EventBus()
    
    // Multiple collectors
    launch {
        eventBus.events.collect { event ->
            println("Collector 1: $event")
        }
    }
    
    launch {
        eventBus.collect { event ->
            println("Collector 2: $event")
        }
    }
    
    delay(100L)
    eventBus.sendEvent("Event 1")
    eventBus.sendEvent("Event 2")
    eventBus.sendEvent("Event 3")
    
    delay(100L)
}

// Output:
// Collector 1: Event 1
// Collector 2: Event 1
// Collector 1: Event 2
// Collector 2: Event 2
// Collector 1: Event 3
// Collector 2: Event 3
```

## 11. Practical Example: User Search with Flow

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class UserRepository {
    fun searchUsers(query: String): Flow<User> {
        return flow {
            // Simulate API call
            delay(500L)
            val allUsers = listOf(
                User(1, "Alice", "alice@example.com"),
                User(2, "Bob", "bob@example.com"),
                User(3, "Alexander", "alex@example.com"),
                User(4, "Alice Cooper", "alicec@example.com")
            )
            
            allUsers
                .filter { it.name.contains(query, ignoreCase = true) }
                .forEach { emit(it) }
        }
    }
}

data class User(val id: Int, val name: String, val email: String)

fun main() = runBlocking {
    val repository = UserRepository()
    
    // Search with 500ms delay between characters
    repository.searchUsers("Alice")
        .debounce(300)      // Wait 300ms for input changes
        .filter { it.name.length > 3 }  // Filter short names
        .map { "${it.name} (${it.email})" }  // Format
        .take(5)            // Max 5 results
        .collect { user ->
            println("Found: $user")
        }
}

// Output:
// Found: Alice (alice@example.com)
// Found: Alice Cooper (alicec@example.com)
```

## Key Takeaways from Lesson 4

1. **Flow is cold** - Only runs when collected
2. **Terminal operations** - `collect`, `toList`, `toSet`, `first`, `count`
3. **Intermediate operators** - `map`, `filter`, `take`, `drop`
4. **Flow composition** - `zip`, `combine` for combining flows
5. **Flattening** - `flattenConcat`, `flatMapConcat`, `flatMapMerge`
6. **Error handling** - `catch` and try-catch blocks
7. **Context preservation** - Flows maintain their execution context
8. **StateFlow** - For state management (like LiveData)
9. **SharedFlow** - For broadcasting events to multiple collectors
10. **Operators are lazy** - Composed with minimal overhead

## Practice Exercise

Create a Flow that:
1. Emits numbers 1 to 100 with 100ms delay
2. Filters only even numbers
3. Squares the filtered numbers
4. Only keeps the first 5 results
5. Handles any errors gracefully
6. Collects and prints the results

<details>
<summary>Solution</summary>

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    flow {
        for (i in 1..100) {
            delay(100L)
            emit(i)
        }
    }
    .filter { it % 2 == 0 }        // Filter even numbers
    .map { it * it }                // Square them
    .take(5)                        // Take first 5
    .catch { e ->
        println("Error: ${e.message}")
    }
    .collect { value ->
        println("Result: $value")
    }
}

// Output:
// Result: 4
// Result: 16
// Result: 36
// Result: 64
// Result: 100
```
</details>

---

Next: [Lesson 5: Advanced Coroutines Patterns](lesson-05-advanced-patterns.md)