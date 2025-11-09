# Lesson 6: Real-World Applications and Integration

## 1. Android Architecture Integration

**MVVM with StateFlow and ViewModel**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import androidx.lifecycle.*

// ViewModel responsible for managing and exposing user list UI state.
class UserListViewModel(
    private val userRepository: UserRepository,          // Repository providing user data
    savedStateHandle: SavedStateHandle                   // Used to persist/restore UI state (like search query)
) : ViewModel() {

    // Backing StateFlow for UI state (Loading, Success, Error)
    private val _state = MutableStateFlow<UserListState>(UserListState.Loading)
    val state: StateFlow<UserListState> = _state.asStateFlow()   // Exposed as read-only StateFlow

    // SharedFlow for one-time error events (e.g., to show a Snackbar)
    private val _error = MutableSharedFlow<String>()
    val error: SharedFlow<String> = _error.asSharedFlow()

    // Tracks loading progress (e.g., show/hide progress bar)
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    // Search query handling using LiveData from SavedStateHandle.
    // Converts to Flow, adds debounce and distinctUntilChanged for better UX.
    val searchQuery = savedStateHandle.getLiveData<String>("searchQuery", "")
        .asFlow()                                      // Convert LiveData to Flow for reactive operators
        .debounce(300L)                                // Wait 300ms after user stops typing
        .filter { it.length >= 2 || it.isEmpty() }     // Ignore very short queries (less than 2 chars)
        .distinctUntilChanged()                        // Only react when query actually changes
        .flatMapLatest { query ->                      // Cancel old searches when a new one starts
            if (query.isEmpty()) {
                // Empty query → emit empty user list
                flowOf(UserListState.Success(emptyList()))
            } else {
                // Non-empty query → perform search
                userRepository.searchUsers(query)
                    .map { UserListState.Success(it) }  // Map successful result to Success state
                    .catch { throwable ->               // Handle errors during search
                        _error.emit(throwable.message ?: "Unknown error")
                        emit(UserListState.Error(throwable.message ?: "Error"))
                    }
                    .onStart { _isLoading.value = true }        // Set loading flag before search starts
                    .onCompletion { _isLoading.value = false }  // Reset loading flag after search ends
            }
        }
        // Convert to StateFlow so it always holds the latest state and can be observed from UI
        .stateIn(
            viewModelScope,                              // Scope tied to ViewModel lifecycle
            SharingStarted.WhileSubscribed(5000L),        // Keeps flow active while UI is observing
            UserListState.Loading                         // Initial state
        )

    // Function to manually refresh the full user list (e.g., pull-to-refresh)
    fun refreshUsers() {
        viewModelScope.launch {
            _state.value = UserListState.Loading          // Show loading state
            try {
                val users = userRepository.getAllUsers()  // Fetch from repository
                _state.value = UserListState.Success(users)
            } catch (e: Exception) {
                // Handle errors gracefully
                _state.value = UserListState.Error(e.message ?: "Refresh failed")
                _error.emit(e.message ?: "Unknown error")
            }
        }
    }
}

// Represents the possible UI states for the user list screen.
sealed class UserListState {
    object Loading : UserListState()                      // UI is loading data
    data class Success(val users: List<User>) : UserListState() // Successfully loaded users
    data class Error(val message: String) : UserListState()      // Error occurred
}

// Simple data model representing a user.
data class User(val id: Int, val name: String, val email: String)

// Repository that handles user data from remote and local sources, with in-memory caching.
class UserRepository {
    private val remoteDataSource = UserRemoteDataSource()   // Simulates network source
    private val localDataSource = UserLocalDataSource()     // Simulates local database
    private val cache = mutableMapOf<Int, User>()           // In-memory cache

    // Searches for users by name, emitting results from cache, then remote, then local fallback.
    suspend fun searchUsers(query: String): Flow<List<User>> {
        return flow {
            // 1️⃣ Emit cached results immediately if available
            val cached = cache.values.filter { it.name.contains(query, ignoreCase = true) }
            if (cached.isNotEmpty()) {
                emit(cached)
            }

            // 2️⃣ Try fetching fresh results from remote source
            try {
                val remote = remoteDataSource.searchUsers(query)
                // Update cache and local database
                remote.forEach { cache[it.id] = it }
                localDataSource.saveUsers(remote)
                // Emit updated cached results
                emit(cache.values.filter { it.name.contains(query, ignoreCase = true) })
            } catch (e: Exception) {
                // 3️⃣ If network fails, fall back to local database
                val local = localDataSource.getUsersByName(query)
                emit(local)
                // Rethrow exception so ViewModel can handle it
                throw e
            }
        }
    }
}
```

## 2. Web Application Backend

**Spring Boot with Coroutines**

```kotlin
import org.springframework.web.bind.annotation.*
import org.springframework.http.ResponseEntity
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import org.springframework.web.bind.annotation.*
import org.springframework.stereotype.Service
import org.springframework.data.mongodb.core.mapping.Document
import org.springframework.data.mongodb.repository.ReactiveMongoRepository
import org.springframework.stereotype.Repository

@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {
    
    @GetMapping
    suspend fun getAllUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): ResponseEntity<List<User>> {
        return try {
            val users = userService.getUsersPaginated(page, size)
            ResponseEntity.ok(users)
        } catch (e: Exception) {
            ResponseEntity.internalServerError().build()
        }
    }
    
    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: String): ResponseEntity<User> {
        return try {
            userService.getUserById(id)?.let { user ->
                ResponseEntity.ok(user)
            } ?: ResponseEntity.notFound().build()
        } catch (e: Exception) {
            ResponseEntity.internalServerError().build()
        }
    }
    
    @PostMapping
    suspend fun createUser(@RequestBody userRequest: UserRequest): ResponseEntity<User> {
        return try {
            val user = userService.createUser(userRequest)
            ResponseEntity.ok(user)
        } catch (e: Exception) {
            ResponseEntity.badRequest().build()
        }
    }
}

@Service
class UserService(
    private val userRepository: UserRepository,
    private val cacheManager: CacheManager
) {
    suspend fun getUsersPaginated(page: Int, size: Int): List<User> {
        return withContext(Dispatchers.IO) {
            // Use cache for popular queries
            val cacheKey = "users:$page:$size"
            cacheManager.get<User>(cacheKey) ?: run {
                val users = userRepository.findAll()
                    .skip(page.toLong() * size)
                    .take(size)
                    .asFlow()
                    .toList()
                cacheManager.put(cacheKey, users, Duration.ofMinutes(5))
                users
            }
        }
    }
    
    suspend fun getUserById(id: String): User? {
        return withContext(Dispatchers.IO) {
            // Try cache first
            val cacheKey = "user:$id"
            cacheManager.get<User>(cacheKey) ?: run {
                val user = userRepository.findById(id)
                if (user != null) {
                    cacheManager.put(cacheKey, user, Duration.ofMinutes(10))
                }
                user
            }
        }
    }
    
    suspend fun createUser(request: UserRequest): User {
        return withContext(Dispatchers.IO) {
            val user = User(
                id = UUID.randomUUID().toString(),
                name = request.name,
                email = request.email,
                createdAt = Instant.now()
            )
            
            // Save to database
            userRepository.save(user).block()
            
            // Invalidate related caches
            cacheManager.evict("users:*")
            
            user
        }
    }
}

@Document
data class User(
    val id: String,
    val name: String,
    val email: String,
    val createdAt: Instant,
    val updatedAt: Instant? = null
)

data class UserRequest(val name: String, val email: String)

@Repository
interface UserRepository : ReactiveMongoRepository<User, String>
```

## 3. Game Development Pattern

**Game Loop with Coroutines**

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis
import java.util.concurrent.ConcurrentLinkedQueue

class GameEngine {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    private val gameState = GameState()
    private val eventQueue = ConcurrentLinkedQueue<GameEvent>()
    
    private val _gameLoopJob = scope.async {
        runGameLoop()
    }
    
    private suspend fun runGameLoop() {
        val targetFps = 60
        val frameTime = 1000L / targetFps
        
        while (gameState.isRunning) {
            val frameStart = System.currentTimeMillis()
            
            // Process events
            processEvents()
            
            // Update game state
            updateGame(frameTime)
            
            // Render
            render()
            
            // Control frame rate
            val frameDuration = System.currentTimeMillis() - frameStart
            val sleepTime = frameTime - frameDuration
            if (sleepTime > 0) {
                delay(sleepTime)
            }
        }
    }
    
    private fun processEvents() {
        while (true) {
            val event = eventQueue.poll() ?: break
            when (event) {
                is PlayerMoveEvent -> handlePlayerMove(event)
                is KeyPressEvent -> handleKeyPress(event)
                is MouseClickEvent -> handleMouseClick(event)
            }
        }
    }
    
    private fun updateGame(deltaTime: Long) {
        gameState.entities.forEach { entity ->
            entity.update(deltaTime)
        }
        
        // Collision detection
        detectCollisions()
        
        // AI updates
        gameState.aiEntities.forEach { ai ->
            ai.update(deltaTime)
        }
    }
    
    private fun render() {
        // Render all entities
        gameState.entities.forEach { it.render() }
    }
    
    private fun detectCollisions() {
        // Parallel collision detection for performance
        scope.launch {
            gameState.entities
                .asFlow()
                .concurrentMapNotNull { entity ->
                    if (entity.collidesWith(gameState.player)) {
                        entity to gameState.player
                    } else null
                }
                .collect { (entityA, entityB) ->
                    handleCollision(entityA, entityB)
                }
        }
    }
    
    fun queueEvent(event: GameEvent) {
        eventQueue.offer(event)
    }
    
    fun stop() {
        gameState.isRunning = false
        scope.cancel()
    }
    
    fun isRunning() = gameState.isRunning
}

data class GameState(
    val player: Player = Player(100, 100),
    val entities: MutableList<Entity> = mutableListOf(),
    val aiEntities: MutableList<AIEntity> = mutableListOf(),
    var isRunning: Boolean = true
)

data class Player(var x: Int, var y: Int) {
    var health: Int = 100
    var isAlive: Boolean = true
}

sealed class GameEvent
data class PlayerMoveEvent(val dx: Int, val dy: Int) : GameEvent()
data class KeyPressEvent(val key: Char) : GameEvent()
data class MouseClickEvent(val x: Int, val y: Int) : GameEvent()

// Usage
fun main() = runBlocking {
    val game = GameEngine()
    
    // Simulate game events
    game.queueEvent(PlayerMoveEvent(10, 0))
    game.queueEvent(KeyPressEvent('W'))
    game.queueEvent(MouseClickEvent(150, 200))
    
    // Run for 5 seconds
    delay(5000L)
    game.stop()
}
```

## 4. Microservices Communication

**Service Mesh with Coroutines**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import retrofit2.*
import retrofit2.http.*

// Service Interfaces
interface UserService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): Response<User>
    
    @GET("users")
    suspend fun getUsers(@Query("page") page: Int): Response<List<User>>
}

interface OrderService {
    @GET("orders/user/{userId}")
    suspend fun getOrders(@Path("userId") userId: String): Response<List<Order>>
}

interface InventoryService {
    @GET("inventory/{productId}")
    suspend fun getInventory(@Path("productId") productId: String): Response<Inventory>
}

interface NotificationService {
    @POST("notifications/send")
    suspend fun sendNotification(@Body notification: Notification): Response<Unit>
}

data class User(
    val id: String,
    val name: String,
    val email: String
)

data class Order(
    val id: String,
    val userId: String,
    val productId: String,
    val status: String
)

data class Inventory(
    val productId: String,
    val quantity: Int,
    val price: Double
)

data class Notification(
    val userId: String,
    val message: String,
    val type: String
)

// Customer Service Aggregator
class CustomerService(
    private val userService: UserService,
    private val orderService: OrderService,
    private val inventoryService: InventoryService,
    private val notificationService: NotificationService
) {
    suspend fun getCustomerDashboard(userId: String): CustomerDashboard {
        return supervisorScope {
            // Parallel data fetching with error handling
            val userDeferred = async(Dispatchers.IO + supervisor) {
                runCatching { userService.getUser(userId) }
                    .getOrElse { Response.error(404, ResponseBody.create(null, "User not found")) }
            }
            
            val ordersDeferred = async(Dispatchers.IO + supervisor) {
                runCatching { orderService.getOrders(userId) }
                    .getOrElse { Response.success(emptyList()) }
            }
            
            // Wait for all data
            val userResponse = userDeferred.await()
            val ordersResponse = ordersDeferred.await()
            
            if (!userResponse.isSuccessful) {
                throw UserNotFoundException("User $userId not found")
            }
            
            val user = userResponse.body()!!
            val orders = ordersResponse.body()?.filter { it.status == "PENDING" } ?: emptyList()
            
            // Fetch inventory for pending orders
            val inventoryDeferred = orders.map { order ->
                async(Dispatchers.IO) {
                    runCatching { inventoryService.getInventory(order.productId) }
                        .getOrElse { Response.error(404, ResponseBody.create(null, "Product not found")) }
                }
            }
            
            val inventory = inventoryDeferred.awaitAll()
                .filter { it.isSuccessful }
                .mapNotNull { it.body() }
                .associateBy { it.productId }
            
            // Calculate dashboard metrics
            val totalValue = orders.sumOf { order ->
                inventory[order.productId]?.price ?: 0.0
            }
            
            // Send notification
            launch(Dispatchers.IO) {
                runCatching {
                    notificationService.sendNotification(
                        Notification(
                            userId = userId,
                            message = "Dashboard loaded. Total pending value: $totalValue",
                            type = "DASHBOARD"
                        )
                    )
                }.onFailure { e ->
                    println("Failed to send notification: ${e.message}")
                }
            }
            
            CustomerDashboard(
                user = user,
                pendingOrders = orders,
                totalPendingValue = totalValue,
                availableProducts = inventory.values.toList()
            )
        }
    }
    
    // Batch processing for multiple customers
    suspend fun processMultipleCustomers(userIds: List<String>): List<ProcessingResult> {
        return withContext(Dispatchers.Default) {
            userIds.asFlow()
                .map { userId ->
                    async {
                        try {
                            val dashboard = getCustomerDashboard(userId)
                            ProcessingResult.Success(userId, dashboard)
                        } catch (e: Exception) {
                            ProcessingResult.Failure(userId, e.message ?: "Unknown error")
                        }
                    }
                }
                .buffer(10) // Process 10 customers concurrently
                .map { it.await() }
                .toList()
        }
    }
}

sealed class ProcessingResult {
    data class Success(val userId: String, val dashboard: CustomerDashboard) : ProcessingResult()
    data class Failure(val userId: String, val error: String) : ProcessingResult()
}

data class CustomerDashboard(
    val user: User,
    val pendingOrders: List<Order>,
    val totalPendingValue: Double,
    val availableProducts: List<Inventory>
)

class UserNotFoundException(message: String) : Exception(message)
```

## 5. Testing Complex Coroutine Scenarios

**Comprehensive Test Suite**

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import org.junit.Test
import org.junit.Assert.*
import org.mockito.Mockito.*
import org.mockito.ArgumentMatchers.anyString

class UserViewModelTest {
    @Test
    fun `loadUser when successful should update state`() = runTest {
        // Arrange
        val userService = mock(UserService::class.java)
        val viewModel = UserViewModel(userService)
        val expectedUser = User("1", "Test User", "test@example.com")
        
        `when`(userService.getUser("1")).thenReturn(expectedUser)
        
        // Act
        viewModel.loadUser("1")
        advanceTimeBy(1000L) // Wait for loading
        val state = viewModel.user.value
        
        // Assert
        assertEquals("1", state?.id)
        assertEquals("Test User", state?.name)
        assertFalse(viewModel.isLoading.value)
    }
    
    @Test
    fun `loadUser when service fails should update error state`() = runTest {
        // Arrange
        val userService = mock(UserService::class.java)
        val viewModel = UserViewModel(userService)
        val exception = RuntimeException("Network error")
        
        `when`(userService.getUser("1")).thenThrow(exception)
        
        // Act
        viewModel.loadUser("1")
        advanceTimeBy(1000L)
        
        // Assert
        assertTrue(viewModel.error.value.contains("Network error"))
    }
    
    @Test
    fun `multiple concurrent requests should respect user actions`() = runTest {
        // Arrange
        val userService = mock(UserService::class.java)
        val viewModel = UserViewModel(userService)
        val user1 = User("1", "User 1", "user1@example.com")
        val user2 = User("2", "User 2", "user2@example.com")
        
        var callCount = 0
        `when`(userService.getUser(anyString())).thenAnswer {
            callCount++
            delay(500L) // Simulate network delay
            if (callCount == 1) user1 else user2
        }
        
        // Act - Make two concurrent requests
        val job1 = launch { viewModel.loadUser("1") }
        val job2 = launch { viewModel.loadUser("2") }
        job1.join()
        job2.join()
        
        // Assert
        val finalState = viewModel.user.value
        // Should have the result from the second call
        assertEquals("User 2", finalState?.name)
    }
}

class FlowTest {
    @Test
    fun `searchUsers should debounce input`() = runTest {
        // Arrange
        val repository = mock(UserRepository::class.java)
        val searchFlow = repository.searchUsers("test")
        
        // Act
        searchFlow.collect()
        advanceTimeBy(300L) // Wait for debounce
        
        // Assert
        verify(repository, atMost(1)).searchUsers("test")
    }
}

class CircuitBreakerTest {
    @Test
    fun `circuit breaker should open after threshold failures`() = runTest {
        // Arrange
        val circuitBreaker = CircuitBreaker(failureThreshold = 3, resetTimeoutMillis = 1000L)
        var callCount = 0
        
        // Act & Assert
        repeat(5) { i ->
            val shouldThrow = i < 4
            if (shouldThrow) {
                assertThrows(CircuitOpenException::class.java) {
                    runTest {
                        circuitBreaker.execute {
                            throw RuntimeException("Service unavailable")
                        }
                    }
                }
            } else {
                // After threshold, should throw CircuitOpenException
                assertThrows(CircuitOpenException::class.java) {
                    runTest {
                        circuitBreaker.execute {
                            "Success"
                        }
                    }
                }
            }
        }
    }
}
```

## 6. Performance Monitoring and Metrics

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.metrics.*
import kotlin.system.measureTimeMillis
import java.time.Instant
import java.util.concurrent.ConcurrentHashMap

class CoroutineMetrics {
    private val metrics = ConcurrentHashMap<String, Metric>()
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun <T> measureOperation(
        operationName: String,
        block: suspend CoroutineScope.() -> T
    ): T {
        val startTime = System.currentTimeMillis()
        val startCpuTime = Thread.currentThread().cpuTime
        
        val result = runBlocking {
            block()
        }
        
        val endTime = System.currentTimeMillis()
        val endCpuTime = Thread.currentThread().cpuTime
        val duration = endTime - startTime
        val cpuDuration = endCpuTime - startCpuTime
        
        updateMetric(operationName, duration, cpuDuration, true)
        return result
    }
    
    private fun updateMetric(
        operationName: String,
        duration: Long,
        cpuDuration: Long,
        success: Boolean
    ) {
        val metric = metrics.getOrPut(operationName) {
            Metric(
                name = operationName,
                count = 0,
                totalDuration = 0L,
                totalCpuTime = 0L,
                successCount = 0,
                failureCount = 0
            )
        }
        
        metric.count++
        metric.totalDuration += duration
        metric.totalCpuTime += cpuDuration
        
        if (success) {
            metric.successCount++
        } else {
            metric.failureCount++
        }
        
        metrics[operationName] = metric
    }
    
    fun getMetrics(): Map<String, Metric> = metrics.toMap()
    
    fun startContinuousMonitoring() {
        scope.launch {
            while (true) {
                delay(30000L) // Every 30 seconds
                reportMetrics()
            }
        }
    }
    
    private suspend fun reportMetrics() {
        val metricsReport = metrics.values.joinToString("\n") { metric ->
            val avgDuration = if (metric.count > 0) {
                metric.totalDuration / metric.count
            } else 0
            
            val successRate = if (metric.count > 0) {
                (metric.successCount.toFloat() / metric.count * 100).format(2)
            } else 0.0
            
            """
            |${metric.name}:
            |  Count: ${metric.count}
            |  Avg Duration: ${avgDuration}ms
            |  Success Rate: ${successRate}%
            |  CPU Time: ${metric.totalCpuTime}ns
            """.trimMargin()
        }
        
        println("=== Coroutine Metrics ===\n$metricsReport")
    }
}

data class Metric(
    val name: String,
    var count: Int,
    var totalDuration: Long,
    var totalCpuTime: Long,
    var successCount: Int,
    var failureCount: Int
)

// Usage
fun main() = runBlocking {
    val metrics = CoroutineMetrics()
    
    val results = List(10) { index ->
        metrics.measureOperation("user-fetch-$index") {
            delay(100L)
            "User $index"
        }
    }
    
    // Start monitoring
    metrics.startContinuousMonitoring()
    
    // Keep running for monitoring
    delay(60000L)
}
```

## Key Takeaways from Lesson 6

### Android Integration
- **ViewModel with StateFlow** - Reactive state management
- **Repository pattern** - Proper data layer architecture
- **Debounced search** - Efficient user input handling

### Web Development
- **Spring WebFlux** - Reactive web applications
- **ReactiveMongoRepository** - Async database operations
- **Caching strategies** - Performance optimization

### Game Development
- **Game loops** - Smooth frame rate control
- **Event processing** - Input handling
- **Parallel processing** - Performance optimization

### Microservices
- **Parallel fetching** - Efficient data aggregation
- **Error isolation** - Independent service failures
- **Circuit breakers** - Resilience patterns

### Testing
- **TestScope** - Deterministic testing
- **Time advancement** - Fast test execution
- **Mock integration** - Isolated testing

### Monitoring
- **Performance metrics** - Operation tracking
- **Resource usage** - CPU and time analysis
- **Real-time monitoring** - Continuous observation

---

Next: [Quick Reference Guide](quick-reference.md)
