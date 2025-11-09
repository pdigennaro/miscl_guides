# Flow: Complete Guide to Reactive Android Architecture

## Table of Contents
1. [LiveData vs Flow Overview](#livedata-vs-flow-overview)
2. [Jetpack Compose + Flow Reactive Patterns](#jetpack-compose--flow-reactive-patterns)
3. [Repository Layer Implementation](#repository-layer-implementation)
4. [Complete Examples](#complete-examples)
5. [Best Practices](#best-practices)
6. [Migration Strategy](#migration-strategy)

---

## LiveData vs Flow Overview

### LiveData
- **Purpose**: Lifecycle-aware observable data holder (Android Architecture Components)
- **Context**: Primarily used in Android development
- **Key Features**:
  - Lifecycle-aware (automatically stops observing when Activity/Fragment is destroyed)
  - Must be used on the main thread
  - Simple observable pattern
  - Part of Android Jetpack

### Flow
- **Purpose**: Cold reactive streams (Kotlin Coroutines)
- **Context**: General Kotlin programming, not limited to Android
- **Key Features**:
  - Cold streams (only emit when collected)
  - Backpressure support
  - Can be used on any thread with proper dispatchers
  - More powerful and flexible
  - Terminal operators like `toList()`, `first()`, `single()`
  - Can be hot or cold depending on implementation

### When to Use Each

#### Use **LiveData** when:
- Working with Android UI components
- Need automatic lifecycle management
- Simple data observation in Activities/Fragments
- Integration with ViewModel
- When you want to ensure UI components don't crash due to lifecycle issues

#### Use **Flow** when:
- Building reactive data pipelines
- Need to perform transformations, filtering, mapping
- Working with multiple data sources
- Need backpressure handling
- Want to test easily (flows are more testable)
- Building general Kotlin applications (not just Android)
- Need to emit multiple values over time

---

## Jetpack Compose + Flow Reactive Patterns

### 1. ViewModel → Flow Pattern

```kotlin
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    private val _userInput = MutableStateFlow("")
    val userInput: StateFlow<String> = _userInput.asStateFlow()
    
    init {
        loadData()
    }
    
    private fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = repository.getData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun onUserInputChanged(input: String) {
        _userInput.value = input
    }
}

// UI State sealed class
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: String) : UiState()
    data class Error(val message: String) : UiState()
}
```

### 2. Compose → Flow Collection

#### Using `collectAsState()` (Recommended for simple cases)

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    val userInput by viewModel.userInput.collectAsState()
    
    when (val state = uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> SuccessScreen(data = state.data, userInput = userInput)
        is UiState.Error -> ErrorScreen(message = state.message)
    }
}
```

#### Using `collectAsStateWithLifecycle()` (Recommended for production)

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val userInput by viewModel.userInput.collectAsStateWithLifecycle()
    
    // Your UI composition
}
```

#### Manual Flow Collection for Complex Scenarios

```kotlin
@Composable
fun ComplexScreen(viewModel: MyViewModel = viewModel()) {
    var selectedItem by remember { mutableStateOf<String?>(null) }
    
    // Collect multiple flows
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val userData by viewModel.userData.collectAsStateWithLifecycle()
    
    // You can also use LaunchedEffect for one-time collections
    LaunchedEffect(viewModel) {
        viewModel.events.collect { event ->
            when (event) {
                is NavigationEvent -> navigateToScreen(event.route)
                is SnackBarEvent -> showSnackBar(event.message)
            }
        }
    }
    
    // Your UI composition
}
```

### 3. Advanced Patterns

#### StateFlow with Compose State

```kotlin
class MyViewModel : ViewModel() {
    // Using StateFlow for UI state
    private val _uiState = MutableStateFlow(UiState())
    val uiState = _uiState.asStateFlow()
    
    // Expose as Compose State
    val uiStateAsComposeState: State<UiState> = _uiState.asStateFlow()
        .asStateFlow()  // Convert to Compose State
}

@Composable
fun Screen(viewModel: MyViewModel = viewModel()) {
    val uiState = viewModel.uiStateAsComposeState
    // Can directly use uiState.value or destructure
}
```

#### SharedFlow for Events

```kotlin
class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>()
    val events = _events.asSharedFlow()
    
    fun performAction() {
        viewModelScope.launch {
            _events.emit(UiEvent.ShowMessage("Action completed!"))
        }
    }
}

@Composable
fun EventHandlingScreen(viewModel: MyViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // Handle one-time events
    LaunchedEffect(viewModel) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowMessage -> showToast(event.message)
                is UiEvent.Navigate -> navigateTo(event.destination)
            }
        }
    }
}
```

---

## Repository Layer Implementation

### 1. Repository Interface with Flow

```kotlin
interface UserRepository {
    fun getAllUsers(): Flow<List<User>>
    fun searchUsers(query: String): Flow<List<User>>
    fun getUserById(id: String): Flow<User?>
    suspend fun getUserByIdSingle(id: String): User?
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
    fun observeUser(id: String): Flow<User>
}
```

### 2. Repository Implementation

```kotlin
class UserRepositoryImpl(
    private val userDao: UserDao,
    private val apiService: UserApiService,
    private val networkMonitor: NetworkMonitor
) : UserRepository {
    
    override fun getAllUsers(): Flow<List<User>> = 
        userDao.getAllUsers()
            .map { userEntities ->
                userEntities.map { it.toDomain() }
            }
            .catch { e ->
                // Handle database errors
                emit(emptyList())
            }
    
    override fun searchUsers(query: String): Flow<List<User>> = 
        userDao.searchUsers("%$query%")
            .map { userEntities ->
                userEntities.map { it.toDomain() }
            }
            .debounce(100) // Debounce to avoid too many queries
            .distinctUntilChanged()
    
    override fun getUserById(id: String): Flow<User?> =
        userDao.getUserById(id)
            .map { it?.toDomain() }
            .catch { e ->
                // Handle errors but don't crash
                emit(null)
            }
    
    override suspend fun getUserByIdSingle(id: String): User? {
        return try {
            userDao.getUserByIdSingle(id)?.toDomain()
        } catch (e: Exception) {
            null
        }
    }
    
    override suspend fun saveUser(user: User) {
        try {
            val userEntity = user.toEntity()
            
            // If online, try to sync with API
            if (networkMonitor.isOnline()) {
                apiService.saveUser(user).let { apiUser ->
                    userEntity.apply {
                        apiId = apiUser.id
                        updatedAt = System.currentTimeMillis()
                        isSynced = true
                    }
                }
            } else {
                // Mark as not synced for later synchronization
                userEntity.isSynced = false
            }
            
            userDao.upsert(userEntity)
        } catch (e: Exception) {
            // Handle network/save errors
            userDao.upsert(user.toEntity().apply { isSynced = false })
        }
    }
    
    override suspend fun deleteUser(id: String) {
        try {
            if (networkMonitor.isOnline()) {
                apiService.deleteUser(id)
            }
            userDao.deleteById(id)
        } catch (e: Exception) {
            // Mark for deletion when online
            userDao.markForDeletion(id)
        }
    }
    
    override fun observeUser(id: String): Flow<User> =
        userDao.observeUser(id)
            .filterNotNull()
            .map { it.toDomain() }
            .distinctUntilChanged()
}
```

### 3. Database Layer (Room + Flow)

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>
    
    @Query("SELECT * FROM users WHERE name LIKE '%' || :query || '%' OR email LIKE '%' || :query || '%'")
    fun searchUsers(query: String): Flow<List<UserEntity>>
    
    @Query("SELECT * FROM users WHERE id = :id")
    fun getUserById(id: String): Flow<UserEntity?>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserByIdSingle(id: String): UserEntity?
    
    @Upsert
    suspend fun upsert(user: UserEntity)
    
    @Delete
    suspend fun delete(user: UserEntity)
    
    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteById(id: String)
    
    @Query("UPDATE users SET isDeleted = 1 WHERE id = :id")
    suspend fun markForDeletion(id: String)
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(users: List<UserEntity>)
    
    // Reactive query for observing single user
    @Query("SELECT * FROM users WHERE id = :id")
    fun observeUser(id: String): Flow<UserEntity?>
}

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey
    val id: String = UUID.randomUUID().toString(),
    val name: String,
    val email: String,
    val age: Int?,
    val apiId: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis(),
    val isSynced: Boolean = true,
    val isDeleted: Boolean = false
)

// Extension functions
fun UserEntity.toDomain(): User = User(
    id = this.id,
    name = this.name,
    email = this.email,
    age = this.age,
    updatedAt = this.updatedAt
)

fun User.toEntity(): UserEntity = UserEntity(
    id = this.id,
    name = this.name,
    email = this.email,
    age = this.age,
    apiId = this.apiId,
    createdAt = this.createdAt,
    updatedAt = this.updatedAt
)
```

### 4. Network Layer (Retrofit + Flow)

```kotlin
interface UserApiService {
    @GET("users")
    suspend fun getUsers(): List<ApiUser>
    
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): ApiUser
    
    @POST("users")
    suspend fun saveUser(@Body user: User): ApiUser
    
    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: String, @Body user: User): ApiUser
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String)
    
    // Flow-based alternative
    @GET("users")
    fun getUsersFlow(): Flow<List<ApiUser>>
    
    @GET("users/{id}")
    fun getUserFlow(@Path("id") id: String): Flow<ApiUser>
}

data class ApiUser(
    val id: String,
    val name: String,
    val email: String,
    val age: Int?
)

// Extension to convert API user to domain
fun ApiUser.toDomain(): User = User(
    id = this.id,
    name = this.name,
    email = this.email,
    age = this.age,
    apiId = this.id,
    createdAt = System.currentTimeMillis(),
    updatedAt = System.currentTimeMillis()
)
```

### 5. Domain Models

```kotlin
data class User(
    val id: String = UUID.randomUUID().toString(),
    val name: String,
    val email: String,
    val age: Int? = null,
    val apiId: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
) {
    val displayName: String
        get() = if (name.isNotBlank()) name else email
    
    fun isValid(): Boolean = name.isNotBlank() && email.isValidEmail()
}

private fun String.isValidEmail(): Boolean {
    return android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()
}
```

---

## Complete Examples

### Complete User List Example

```kotlin
// ViewModel
class UserListViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(UserListUiState())
    val uiState: StateFlow<UserListUiState> = _uiState.asStateFlow()
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    init {
        loadUsers()
        observeSearchQuery()
    }
    
    fun onSearchQueryChanged(query: String) {
        _searchQuery.value = query
    }
    
    fun onUserSelected(user: User) {
        viewModelScope.launch {
            _uiState.update { it.copy(selectedUser = user) }
        }
    }
    
    private fun observeSearchQuery() {
        viewModelScope.launch {
            searchQuery
                .debounce(300)
                .distinctUntilChanged()
                .flatMapLatest { query ->
                    if (query.isBlank()) {
                        flowOf(emptyList())
                    } else {
                        userRepository.searchUsers(query)
                    }
                }
                .catch { e ->
                    _uiState.update { it.copy(error = e.message) }
                }
                .collect { users ->
                    _uiState.update { it.copy(users = users, isLoading = false) }
                }
        }
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val users = userRepository.getAllUsers()
                _uiState.update { it.copy(users = users, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}

// Data class
data class UserListUiState(
    val users: List<User> = emptyList(),
    val selectedUser: User? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

// Compose Screen
@Composable
fun UserListScreen(
    viewModel: UserListViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val searchQuery by viewModel.searchQuery.collectAsStateWithLifecycle()
    
    Column {
        SearchBar(
            query = searchQuery,
            onQueryChange = viewModel::onSearchQueryChanged
        )
        
        when {
            uiState.isLoading -> LoadingIndicator()
            uiState.error != null -> ErrorMessage(uiState.error!!)
            else -> UserList(
                users = uiState.users,
                onUserClick = viewModel::onUserSelected
            )
        }
    }
}
```

---

## Best Practices

### ✅ **DO**
- Use `StateFlow` for UI state
- Use `SharedFlow` for events
- Use `collectAsStateWithLifecycle()` for production
- Keep ViewModel logic pure and testable
- Use sealed classes for UI state
- Handle loading states explicitly
- Implement proper error handling
- Use repository pattern for data access
- Cache data locally for offline support
- Test your flows thoroughly

### ❌ **DON'T**
- Use `GlobalScope` in ViewModel
- Expose MutableStateFlow directly
- Forget to use `viewModelScope`
- Block the main thread
- Ignore lifecycle considerations
- Mix LiveData and Flow unnecessarily
- Skip error handling in repositories
- Create memory leaks with flows
- Use `collectAsState()` in production (use lifecycle-aware version)

---

## Migration Strategy

For Android development, the recommended approach is:
- Use **ViewModel → Flow** (data layer)
- **Flow → LiveData** (UI layer bridge) - if needed
- **LiveData → UI observers** (View layer) - for traditional views
- **Flow → Compose State** (Compose layer) - for modern UI

```kotlin
// Recommended pattern
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // Expose to UI as LiveData if needed
    val uiStateLiveData: LiveData<UiState> = _uiState.asLiveData()
}
```

## Flow to LiveData Bridge

Sometimes you need to convert Flow to LiveData, especially during migration or when working with legacy code that expects LiveData. Here are the different ways to create this bridge:

### 1. Basic Flow to LiveData Conversion

```kotlin
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // Simple conversion
    val uiStateLiveData: LiveData<UiState> = _uiState.asLiveData()
    
    // Conversion with custom CoroutineScope
    val scopedLiveData: LiveData<String> = _uiState.map { it.data }
        .asLiveData(viewModelScope) // Use viewModelScope for proper lifecycle
}
```

### 2. Lifecycle-Aware Flow to LiveData

```kotlin
class MyViewModel : ViewModel() {
    private val _data = MutableStateFlow<DataState>(DataState.Empty)
    val data: StateFlow<DataState> = _data.asStateFlow()
    
    // Use viewModelScope to ensure LiveData is cancelled when ViewModel is cleared
    val dataLiveData: LiveData<DataState> = _data
        .asLiveData(viewModelScope) // This is lifecycle-aware
        // Alternative: .asLiveData(Dispatchers.Main) with custom scope
    
    // With transformation
    val transformedDataLiveData: LiveData<String> = _data
        .map { state ->
            when (state) {
                is DataState.Success -> "Data: ${state.data}"
                is DataState.Loading -> "Loading..."
                is DataState.Error -> "Error: ${state.message}"
                else -> "No data"
            }
        }
        .asLiveData(viewModelScope)
}
```

### 3. Repository Pattern with LiveData Bridge

```kotlin
// Repository returns Flow
interface UserRepository {
    fun getUsers(): Flow<List<User>>
    fun getUserById(id: String): Flow<User?>
    suspend fun saveUser(user: User)
}

// ViewModel bridges Flow to LiveData
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _selectedUserId = MutableStateFlow<String?>(null)
    val selectedUserId: StateFlow<String?> = _selectedUserId.asStateFlow()
    
    // Expose Flow as LiveData for legacy components
    val usersLiveData: LiveData<List<User>> = 
        repository.getUsers().asLiveData()
    
    val selectedUserLiveData: LiveData<User?> = 
        repository.getUserById("").asLiveData() // Default empty for transformation
    
    // Transform and expose as LiveData
    val formattedUserName: LiveData<String> = 
        repository.getUsers()
            .map { users ->
                users.firstOrNull()?.name ?: "No user selected"
            }
            .distinctUntilChanged()
            .asLiveData()
    
    fun onUserSelected(userId: String) {
        _selectedUserId.value = userId
    }
}
```

### 4. Complex Flow Transformations with LiveData

```kotlin
class ComplexViewModel : ViewModel() {
    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()
    
    // Combine multiple flows and expose as LiveData
    val searchResults: LiveData<List<SearchResult>> = 
        query
            .debounce(500)
            .distinctUntilChanged()
            .flatMapLatest { searchQuery ->
                if (searchQuery.isBlank()) {
                    flowOf(emptyList())
                } else {
                    repository.search(searchQuery)
                }
            }
            .map { results -> 
                results.sortedBy { it.score }
            }
            .asLiveData(viewModelScope)
    
    // Combine two flows
    val combinedData: LiveData<CombinedState> = 
        combine(
            repository.getUsers(),
            repository.getSettings()
        ) { users, settings ->
            CombinedState(
                userCount = users.size,
                isDarkMode = settings.isDarkMode,
                hasNetwork = settings.hasNetwork
            )
        }
        .asLiveData(viewModelScope)
    
    fun updateQuery(newQuery: String) {
        _query.value = newQuery
    }
}

data class CombinedState(
    val userCount: Int,
    val isDarkMode: Boolean,
    val hasNetwork: Boolean
)
```

### 5. Hot Flow Conversions

```kotlin
class HotFlowViewModel : ViewModel() {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    // Convert SharedFlow to LiveData (use case: broadcasting events)
    val lastEvent: LiveData<Event?> = 
        _events
            .onEach { event -> 
                // Handle side effects
                Log.d("ViewModel", "Event: $event")
            }
            .asLiveData(viewModelScope)
    
    // StateFlow to LiveData
    private val _appState = MutableStateFlow(AppState())
    val appState: StateFlow<AppState> = _appState.asStateFlow()
    
    val appStateLiveData: LiveData<AppState> = _appState.asLiveData()
    
    fun triggerEvent(event: Event) {
        viewModelScope.launch {
            _events.emit(event)
        }
    }
    
    fun updateAppState(newState: AppState) {
        _appState.value = newState
    }
}
```

### 6. Error Handling in LiveData Bridge

```kotlin
class ErrorHandlingViewModel : ViewModel() {
    private val _data = MutableStateFlow<DataState>(DataState.Loading)
    val data: StateFlow<DataState> = _data.asData.asStateFlow()
    
    // Convert Flow to LiveData with error handling
    val safeDataLiveData: LiveData<DataState> = 
        _data
            .catch { exception ->
                // Log error and emit error state
                Log.e("ViewModel", "Error in data flow", exception)
                emit(DataState.Error(exception.message ?: "Unknown error"))
            }
            .asLiveData(viewModelScope)
    
    // Alternative with try-catch wrapper
    val wrappedDataLiveData: LiveData<DataState> = 
        flow {
            try {
                emitAll(_data)
            } catch (e: Exception) {
                emit(DataState.Error(e.message ?: "Error occurred"))
            }
        }
        .asLiveData(viewModelScope)
}
```

### 7. Using with View Binding (Legacy Components)

```kotlin
// For activities/fragments that still use View Binding
class LegacyActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var viewModel: MyViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        viewModel = ViewModelProvider(this)[MyViewModel::class.java]
        
        // Use LiveData from ViewModel (which internally uses Flow)
        viewModel.uiStateLiveData.observe(this) { state ->
            updateUI(state)
        }
    }
    
    private fun updateUI(state: UiState) {
        when (state) {
            is UiState.Loading -> {
                binding.progressBar.visibility = View.VISIBLE
                binding.textView.text = "Loading..."
            }
            is UiState.Success -> {
                binding.progressBar.visibility = View.GONE
                binding.textView.text = "Success: ${state.data}"
            }
            is UiState.Error -> {
                binding.progressBar.visibility = View.GONE
                binding.textView.text = "Error: ${state.message}"
            }
        }
    }
}
```

### 8. Best Practices for Flow to LiveData Bridge

#### ✅ **DO**
```kotlin
// Always use viewModelScope
val liveData = myFlow.asLiveData(viewModelScope)

// Use transformation operators
val transformedLiveData = myFlow
    .map { it.toUiModel() }
    .catch { /* handle errors */ }
    .distinctUntilChanged()
    .asLiveData()

// Use proper error handling
val safeLiveData = myFlow
    .catch { e -> 
        emit(ErrorState(e.message))
    }
    .asLiveData(viewModelScope)
```

#### ❌ **DON'T**
```kotlin
// Don't use GlobalScope
val liveData = myFlow.asLiveData() // ❌ Bad

// Don't expose MutableLiveData directly
val badLiveData = MutableLiveData<String>() // ❌ Bad

// Don't forget error handling
val unsafeLiveData = myFlow.asLiveData() // ❌ No error handling
```

## Modern Recommendation

**Use Flow as your primary reactive stream** and only use LiveData as a bridge to the UI layer when necessary. Flow is more powerful, testable, and follows modern Kotlin coroutines patterns.

---

*This guide provides a comprehensive approach to building reactive Android applications using modern Kotlin coroutines and Jetpack Compose.*