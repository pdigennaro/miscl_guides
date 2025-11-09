# Comprehensive Guide to Hilt and Android Jetpack Compose

*Author: MiniMax Agent*  
*Date: November 9, 2025*

## Table of Contents

1. [Introduction](#introduction)
2. [Hilt Dependency Injection](#hilt-dependency-injection)
   - [What is Hilt?](#what-is-hilt)
   - [Setup and Installation](#setup-and-installation)
   - [Core Concepts](#core-concepts)
   - [Annotations](#annotations)
   - [Modules and Bindings](#modules-and-bindings)
   - [Scopes and Lifecycles](#scopes-and-lifecycles)
   - [Best Practices](#hilt-best-practices)
3. [Jetpack Compose](#jetpack-compose)
   - [What is Jetpack Compose?](#what-is-jetpack-compose)
   - [Setup and Installation](#compose-setup-and-installation)
   - [Core Concepts](#compose-core-concepts)
   - [Composable Functions](#composable-functions)
   - [State Management](#state-management)
   - [Layout System](#layout-system)
   - [Material Design](#material-design)
   - [Animations](#animations)
   - [Navigation](#navigation)
4. [Hilt + Jetpack Compose Integration](#hilt--jetpack-compose-integration)
   - [Dependency Injection in Composables](#dependency-injection-in-composables)
   - [ViewModel Injection](#viewmodel-injection)
   - [Repository Pattern](#repository-pattern)
   - [Application Architecture](#application-architecture)
5. [Practical Examples](#practical-examples)
6. [Performance Considerations](#performance-considerations)
7. [Testing](#testing)
8. [Common Patterns and Anti-Patterns](#common-patterns-and-anti-patterns)
9. [Migration Guide](#migration-guide)
10. [Resources and Further Reading](#resources-and-further-reading)

---

## Introduction

Modern Android development has evolved significantly with the introduction of two game-changing technologies: **Hilt** for dependency injection and **Jetpack Compose** for UI development. This comprehensive guide will walk you through both technologies, their integration, and best practices for building robust Android applications.

### Why These Technologies Matter

- **Hilt** simplifies dependency injection by providing a standard way to handle dependencies across your Android application
- **Jetpack Compose** revolutionizes UI development with a declarative approach, making it easier to build and maintain user interfaces
- Together, they enable a more maintainable, testable, and scalable application architecture

---

## Hilt Dependency Injection

### What is Hilt?

Hilt is Android's recommended dependency injection (DI) library that builds on top of Dagger. It simplifies the setup and usage of Dagger in Android applications by providing:

- Automatic dependency injection for Android classes
- Scope management tied to Android component lifecycles
- Reduced boilerplate code
- Better integration with Android development

### Setup and Installation

#### 1. Add Dependencies

```gradle
// app/build.gradle
plugins {
    id 'dagger.hilt.android.plugin'
    id 'kotlin-kapt'
}

android {
    // ... other configurations
}

dependencies {
    implementation 'com.google.dagger:hilt-android:2.48'
    implementation 'androidx.hilt:hilt-navigation-compose:1.1.0'
    implementation 'androidx.hilt:hilt-work:1.1.0'
    
    // Annotation processor
    kapt 'com.google.dagger:hilt-compiler:2.48'
}
```

#### 2. Application Class Setup

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt will generate the necessary components and dependencies
}
```

### Core Concepts

#### 1. Dependency Injection Fundamentals

Dependency Injection is a design pattern where objects receive their dependencies from external sources rather than creating them internally. This promotes:

- **Loose coupling** between components
- **Better testability** through mocking
- **Single responsibility principle**
- **Reusability** of components

#### 2. Hilt Components

Hilt creates the following components for dependency injection:

- **SingletonComponent**: Lives for the entire application lifetime
- **ActivityComponent**: Tied to Activity lifecycle
- **FragmentComponent**: Tied to Fragment lifecycle
- **ViewComponent**: Tied to View lifecycle
- **ViewWithFragmentComponent**: Tied to View and Fragment combined

### Annotations

#### Core Annotations

```kotlin
@Module
// Defines a module that provides dependencies

@InstallIn(ActivityComponent::class)
// Specifies which Android component to install the module in

@AndroidEntryPoint
// Marks an Android class to be injected

@Singleton
@ActivityScoped
@FragmentScoped
// Defines scope annotations

@Provides
@Binds
// Methods to provide dependencies

@Inject
// Constructor injection marker
```

#### Module Example

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

### Modules and Bindings

#### Using @Binds vs @Provides

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    // Use @Binds for interfaces
    @Binds
    abstract fun bindRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
    
    // Use @Provides for regular classes
    @Provides
    fun provideSharedPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }
}
```

### Scopes and Lifecycles

#### Scope Management

```kotlin
@Singleton
@Component
class SingletonComponent { /* ... */ }

@ActivityScoped
@Subcomponent
class ActivityComponent { /* ... */ }

@FragmentScoped
@Subcomponent
class FragmentComponent { /* ... */ }

// Usage in classes
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    @Inject
    lateinit var userManager: UserManager // Injected as singleton
    
    @Inject
    lateinit var activityService: ActivityService // Injected as activity-scoped
}
```

### Hilt Best Practices

#### 1. Keep Modules Focused
```kotlin
// Good: One module per responsibility
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
}
```

#### 2. Use Constructor Injection
```kotlin
class UserRepository @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase
) {
    // Implementation
}
```

#### 3. Handle External Dependencies
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ExternalModule {
    
    @Provides
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }
}
```

---

## Jetpack Compose

### What is Jetpack Compose?

Jetpack Compose is Android's modern, declarative UI toolkit that simplifies and accelerates UI development. Key features include:

- **Declarative programming**: UI is described as a function of the state
- **Composable functions**: UI components are built using Kotlin functions
- **Built-in Material Design**: Consistent theming and components
- **Animation support**: Built-in animation APIs
- **Interoperability**: Works with existing View system

### Setup and Installation

#### 1. Add Dependencies

```gradle
android {
    buildFeatures {
        compose true
    }
    composeOptions {
        kotlinCompilerExtensionVersion '1.5.8'
    }
    kotlinOptions {
        jvmTarget = '17'
    }
}

dependencies {
    implementation platform('androidx.compose:compose-bom:2024.02.00')
    implementation 'androidx.compose.ui:ui'
    implementation 'androidx.compose.ui:ui-tooling-preview'
    implementation 'androidx.compose.material3:material3'
    implementation 'androidx.activity:activity-compose:1.8.2'
    implementation 'androidx.navigation:navigation-compose:2.7.6'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0'
    
    // Testing
    androidTestImplementation platform('androidx.compose:compose-bom:2024.02.00')
    androidTestImplementation 'androidx.compose.ui:ui-test-junit4'
    debugImplementation 'androidx.compose.ui:ui-tooling'
    debugImplementation 'androidx.compose.ui:ui-test-manifest'
}
```

### Compose Core Concepts

#### 1. Composable Functions

```kotlin
@Composable
fun GreetingCard(name: String) {
    Card(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth()
    ) {
        Text(
            text = "Hello $name!",
            modifier = Modifier.padding(16.dp),
            style = MaterialTheme.typography.h6
        )
    }
}
```

#### 2. Recomposition

Compose automatically recomposes when state changes:

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text(text = "Count: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

### Composables and Layout System

#### 1. Basic Composables

```kotlin
@Composable
fun UserProfileCard(user: User) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = "User Avatar",
                modifier = Modifier
                    .size(60.dp)
                    .clip(CircleShape)
            )
            
            Spacer(modifier = Modifier.width(16.dp))
            
            Column {
                Text(
                    text = user.name,
                    style = MaterialTheme.typography.titleMedium
                )
                Text(
                    text = user.email,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
    }
}
```

#### 2. Layout Modifiers

```kotlin
@Composable
fun LayoutExample() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
            .background(MaterialTheme.colorScheme.background)
    ) {
        Text(
            "Header",
            modifier = Modifier
                .fillMaxWidth()
                .background(MaterialTheme.colorScheme.primaryContainer)
                .padding(16.dp)
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        LazyColumn(
            modifier = Modifier.weight(1f)
        ) {
            items(users) { user ->
                UserItem(user = user)
            }
        }
    }
}
```

### State Management

#### 1. State Types and Patterns

```kotlin
// MutableState for simple local state
@Composable
fun SimpleStateExample() {
    var name by remember { mutableStateOf("") }
    
    TextField(
        value = name,
        onValueChange = { name = it },
        label = { Text("Name") }
    )
}

// StateFlow with collectAsState
@Composable
fun ViewModelStateExample(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> {
            CircularProgressIndicator()
        }
        is UiState.Success -> {
            UserList(users = (uiState as UiState.Success).users)
        }
        is UiState.Error -> {
            Text(
                text = (uiState as UiState.Error).message,
                color = MaterialTheme.colorScheme.error
            )
        }
    }
}

// rememberSaveable for configuration changes
@Composable
fun PersistentStateExample() {
    var counter by rememberSaveable { mutableStateOf(0) }
    
    Text(text = "Counter: $counter")
    
    Button(onClick = { counter++ }) {
        Text("Increment")
    }
}
```

#### 2. Unidirectional Data Flow (UDF)

```kotlin
// State class
sealed class UiState {
    data class UserListState(val users: List<User>) : UiState()
    data class LoadingState(val progress: Float) : UiState()
    data class ErrorState(val message: String) : UiState()
}

// ViewModel with UDF
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.LoadingState(0f))
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.LoadingState(0.5f)
            
            try {
                val users = userRepository.getUsers()
                _uiState.value = UiState.UserListState(users)
            } catch (e: Exception) {
                _uiState.value = UiState.ErrorState(e.message ?: "Unknown error")
            }
        }
    }
}
```

### Material Design and Theming

#### 1. Custom Theme

```kotlin
@Composable
fun MyAppTheme(
    content: @Composable () -> Unit
) {
    val colorScheme = lightColorScheme(
        primary = Purple40,
        onPrimary = Color.White,
        primaryContainer = Purple40,
        onPrimaryContainer = Color.White,
        secondary = Teal40,
        onSecondary = Color.White,
        surface = Color.White,
        onSurface = Color.Black,
        // ... other colors
    )
    
    val typography = Typography(
        headlineLarge = TextStyle(
            fontFamily = FontFamily.Default,
            fontWeight = FontWeight.Bold,
            fontSize = 32.sp
        )
    )
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = typography,
        content = content
    )
}
```

#### 2. Material Design Components

```kotlin
@Composable
fun MaterialComponentsDemo() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        // Buttons
        Button(onClick = { /* Action */ }) {
            Text("Primary Button")
        }
        
        OutlinedButton(onClick = { /* Action */ }) {
            Text("Outlined Button")
        }
        
        TextButton(onClick = { /* Action */ }) {
            Text("Text Button")
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Text Fields
        var text by remember { mutableStateOf("") }
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("Enter text") },
            placeholder = { Text("Placeholder") }
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Cards
        Card {
            Text(
                "This is a card",
                modifier = Modifier.padding(16.dp)
            )
        }
    }
}
```

### Animations

#### 1. Simple Animations

```kotlin
@Composable
fun AnimatedButton() {
    var isPressed by remember { mutableStateOf(false) }
    
    val scale by animateFloatAsState(
        targetValue = if (isPressed) 0.95f else 1f,
        animationSpec = spring(
            stiffness = Spring.StiffnessLow,
            dampingRatio = Spring.DampingRatioMediumBouncy
        )
    )
    
    Button(
        onClick = { isPressed = !isPressed },
        modifier = Modifier.scale(scale)
    ) {
        Text(if (isPressed) "Pressed!" else "Click Me!")
    }
}
```

#### 2. List Animations

```kotlin
@Composable
fun AnimatedList(items: List<String>) {
    LazyColumn {
        items(
            items = items,
            key = { it }
        ) { item ->
            AnimatedVisibility(
                visible = true,
                enter = fadeIn() + slideInVertically()
            ) {
                ListItem(
                    headlineContent = { Text(item) }
                )
            }
        }
    }
}
```

### Navigation

#### 1. Navigation Setup

```kotlin
@HiltAndroidApp
class MyApplication : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            MyAppTheme {
                MyApp()
            }
        }
    }
}

@Composable
fun MyApp() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToProfile = { userId ->
                    navController.navigate("profile/$userId")
                }
            )
        }
        composable(
            route = "profile/{userId}",
            arguments = listOf(
                navArgument("userId") { type = NavType.StringType }
            )
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId") ?: ""
            ProfileScreen(userId = userId)
        }
    }
}
```

---

## Hilt + Jetpack Compose Integration

### Dependency Injection in Composables

#### 1. HiltViewModel

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                val userList = userRepository.getUsers()
                _users.value = userList
            } finally {
                _isLoading.value = false
            }
        }
    }
}

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val users by viewModel.users.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    
    if (isLoading) {
        LoadingScreen()
    } else {
        UserListScreen(users = users)
    }
}
```

#### 2. Singleton Dependencies

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    fun provideUserRepository(
        apiService: ApiService,
        database: AppDatabase
    ): UserRepository {
        return UserRepositoryImpl(apiService, database)
    }
    
    @Provides
    fun provideSettingsRepository(
        sharedPreferences: SharedPreferences
    ): SettingsRepository {
        return SettingsRepositoryImpl(sharedPreferences)
    }
}

// Usage in ViewModel
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    
    private val _settings = MutableStateFlow<Settings>(Settings())
    val settings: StateFlow<Settings> = _settings.asStateFlow()
    
    fun updateTheme(isDarkMode: Boolean) {
        settingsRepository.setDarkMode(isDarkMode)
        _settings.value = settingsRepository.getSettings()
    }
}
```

### Repository Pattern

#### 1. Interface and Implementation

```kotlin
// Repository Interface
interface UserRepository {
    suspend fun getUsers(): List<User>
    suspend fun getUserById(id: String): User?
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
}

// Repository Implementation with DI
class UserRepositoryImpl @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase,
    private val cacheManager: CacheManager
) : UserRepository {
    
    override suspend fun getUsers(): List<User> {
        return try {
            // Try network first
            val users = apiService.getUsers()
            cacheManager.cacheUsers(users)
            database.userDao().insertAll(users)
            users
        } catch (e: Exception) {
            // Fall back to database
            database.userDao().getAll()
        }
    }
    
    override suspend fun getUserById(id: String): User? {
        return try {
            apiService.getUserById(id)
        } catch (e: Exception) {
            database.userDao().getUserById(id)
        }
    }
    
    override suspend fun saveUser(user: User) {
        database.userDao().insert(user)
        try {
            apiService.saveUser(user)
        } catch (e: Exception) {
            // Handle offline scenario
        }
    }
    
    override suspend fun deleteUser(id: String) {
        database.userDao().deleteById(id)
        try {
            apiService.deleteUser(id)
        } catch (e: Exception) {
            // Handle offline scenario
        }
    }
}
```

#### 2. Module Configuration

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindUserRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
    
    @Binds
    abstract fun bindSettingsRepository(
        settingsRepositoryImpl: SettingsRepositoryImpl
    ): SettingsRepository
}
```

### Application Architecture

#### 1. Clean Architecture with Hilt and Compose

```
app/
├── data/
│   ├── local/
│   ├── remote/
│   └── repository/
├── domain/
│   ├── model/
│   ├── repository/
│   └── usecase/
├── presentation/
│   ├── home/
│   ├── profile/
│   └── common/
└── di/
    ├── network/
    ├── database/
    └── repository/
```

#### 2. Layer Implementation

```kotlin
// Domain Layer
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String
)

class GetUsersUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(): Result<List<User>> {
        return try {
            val users = userRepository.getUsers()
            Result.success(users)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// Data Layer
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    suspend fun getAll(): List<User>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: String): User?
    
    @Insert
    suspend fun insert(user: User)
    
    @Delete
    suspend fun delete(user: User)
}

// Presentation Layer
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Initial)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            getUsersUseCase().fold(
                onSuccess = { users ->
                    _uiState.value = UiState.Success(users)
                },
                onFailure = { error ->
                    _uiState.value = UiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }
}

sealed class UiState {
    object Initial : UiState()
    object Loading : UiState()
    data class Success(val users: List<User>) : UiState()
    data class Error(val message: String) : UiState()
}
```

---

## Practical Examples

### 1. User Profile Screen

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val updateUserUseCase: UpdateUserUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<ProfileUiState>(ProfileUiState.Loading)
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = ProfileUiState.Loading
            
            getUserUseCase(userId).fold(
                onSuccess = { user ->
                    _uiState.value = ProfileUiState.Success(user)
                },
                onFailure = { error ->
                    _uiState.value = ProfileUiState.Error(error.message ?: "Failed to load user")
                }
            )
        }
    }
    
    fun updateProfile(updatedUser: User) {
        viewModelScope.launch {
            updateUserUseCase(updatedUser).fold(
                onSuccess = {
                    // Handle success
                    _uiState.value = ProfileUiState.Success(updatedUser)
                },
                onFailure = { error ->
                    _uiState.value = ProfileUiState.Error("Failed to update profile")
                }
            )
        }
    }
}

@Composable
fun ProfileScreen(
    userId: String,
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }
    
    when (uiState) {
        is ProfileUiState.Loading -> {
            ProfileLoadingScreen()
        }
        is ProfileUiState.Success -> {
            ProfileForm(
                user = (uiState as ProfileUiState.Success).user,
                onSave = { updatedUser ->
                    viewModel.updateProfile(updatedUser)
                }
            )
        }
        is ProfileUiState.Error -> {
            ProfileErrorScreen(
                message = (uiState as ProfileUiState.Error).message,
                onRetry = { viewModel.loadUser(userId) }
            )
        }
    }
}

@Composable
fun ProfileForm(
    user: User,
    onSave: (User) -> Unit
) {
    var name by remember(user.name) { mutableStateOf(user.name) }
    var email by remember(user.email) { mutableStateOf(user.email) }
    var isEditing by remember { mutableStateOf(false) }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        ProfileHeader(user = user, isEditing = isEditing)
        
        Spacer(modifier = Modifier.height(24.dp))
        
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Name") },
            readOnly = !isEditing,
            modifier = Modifier.fillMaxWidth()
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") },
            readOnly = !isEditing,
            modifier = Modifier.fillMaxWidth()
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Button(
            onClick = {
                if (isEditing) {
                    onSave(user.copy(name = name, email = email))
                }
                isEditing = !isEditing
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(if (isEditing) "Save" else "Edit Profile")
        }
    }
}
```

### 2. Search Feature with Paging

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val searchResults: StateFlow<PagingData<User>> = 
        _searchQuery
            .debounce(300)
            .distinctUntilChanged()
            .flatMapLatest { query ->
                if (query.isEmpty()) {
                    flowOf(PagingData.empty())
                } else {
                    searchRepository.searchUsers(query)
                }
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = PagingData.empty()
            )
    
    fun updateSearchQuery(query: String) {
        _searchQuery.value = query
    }
}

@Composable
fun SearchScreen(
    viewModel: SearchViewModel = hiltViewModel()
) {
    val searchQuery by viewModel.searchQuery.collectAsState()
    val searchResults by viewModel.searchResults.collectAsLazyPagingItems()
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        SearchBar(
            query = searchQuery,
            onQueryChange = viewModel::updateSearchQuery,
            placeholder = { Text("Search users...") }
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        LazyColumn {
            items(
                items = searchResults,
                key = { it.id }
            ) { user ->
                if (user != null) {
                    UserSearchResult(user = user)
                } else {
                    SearchResultShimmer()
                }
            }
        }
    }
}
```

---

## Performance Considerations

### 1. Compose Performance

#### Recomposition Optimization

```kotlin
// Good: Extract expensive calculations outside recomposition
@Composable
fun UserListWithOptimization(users: List<User>) {
    // Calculate expensive operations outside composable
    val sortedUsers = remember(users) {
        users.sortedBy { it.name }
    }
    
    val groupedUsers = remember(sortedUsers) {
        sortedUsers.groupBy { it.name.first() }
    }
    
    LazyColumn {
        groupedUsers.forEach { (initial, userList) ->
            item {
                SectionHeader(initial = initial)
            }
            items(
                items = userList,
                key = { it.id }
            ) { user ->
                UserItem(user = user)
            }
        }
    }
}

// Bad: Expensive operations in composable body
@Composable
fun PoorPerformanceList(users: List<User>) {
    LazyColumn {
        items(users.sortedBy { it.name }) { // Sorting happens on every recomposition
            UserItem(user = it)
        }
    }
}
```

#### State hoisting for better performance

```kotlin
// Good: Hoist state to parent
@Composable
fun UserProfileCard(
    user: User,
    isExpanded: Boolean,
    onExpandToggle: () -> Unit
) {
    Card(
        modifier = Modifier.clickable { onExpandToggle() }
    ) {
        ProfileContent(user = user, isExpanded = isExpanded)
    }
}

// Usage
@Composable
fun UserListItem(user: User) {
    var isExpanded by remember { mutableStateOf(false) }
    
    UserProfileCard(
        user = user,
        isExpanded = isExpanded,
        onExpandToggle = { isExpanded = !isExpanded }
    )
}
```

### 2. Hilt Performance

#### Module Organization

```kotlin
// Good: Organize modules by feature and scope
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @SingletonScope
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }
    
    @Provides
    @SingletonScope
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .client(okHttpClient)
            .baseUrl(BuildConfig.API_BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}

// Good: Use @Binds for performance
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryBindsModule {
    
    @Binds
    abstract fun bindUserRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
}
```

---

## Testing

### 1. Unit Testing with Hilt

```kotlin
// Test Module
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class]
)
object TestNetworkModule {
    
    private val mockWebServer = MockWebServer()
    
    @Provides
    @Singleton
    fun provideMockRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}

// ViewModel Test
@ExtendWith(AndroidJUnit4::class)
@HiltAndroidTest
class UserViewModelTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @Inject
    @field:Mock
    lateinit var userRepository: UserRepository
    
    private lateinit var viewModel: UserViewModel
    
    @Before
    fun setup() {
        hiltRule.inject()
        viewModel = UserViewModel(userRepository)
    }
    
    @Test
    fun `loadUsers should update UI state correctly`() = runTest {
        // Given
        val mockUsers = listOf(
            User("1", "John", "john@example.com", ""),
            User("2", "Jane", "jane@example.com", "")
        )
        every { userRepository.getUsers() } returns mockUsers
        
        // When
        viewModel.loadUsers()
        
        // Then
        val uiState = viewModel.uiState.first()
        assert(uiState is UiState.Success)
        assertEquals(mockUsers, (uiState as UiState.Success).users)
    }
}
```

### 2. Compose Testing

```kotlin
// Test Rule
@get:Rule
val composeTestRule = createAndroidComposeRule<MainActivity>()

// Basic Test
@Test
fun `UserList displays users correctly`() {
    val testUsers = listOf(
        User("1", "John", "john@example.com", ""),
        User("2", "Jane", "jane@example.com", "")
    )
    
    composeTestRule.setContent {
        UserList(users = testUsers)
    }
    
    composeTestRule
        .onNodeWithText("John")
        .assertIsDisplayed()
    
    composeTestRule
        .onNodeWithText("Jane")
        .assertIsDisplayed()
}

// Test with ViewModel
@HiltAndroidTest
class UserListScreenTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun `UserListScreen shows loading then content`() = runTest {
        composeTestRule.setContent {
            UserListScreen()
        }
        
        // Check loading state
        composeTestRule
            .onNodeWithTag("LoadingIndicator")
            .assertIsDisplayed()
        
        // Wait for data to load
        advanceTimeBy(1000)
        
        // Check content
        composeTestRule
            .onNodeWithText("John")
            .assertIsDisplayed()
    }
}
```

### 3. Integration Testing

```kotlin
@HiltAndroidTest
class UserProfileIntegrationTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Inject
    @field:Mock
    lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun `user can edit and save profile`() = runTest {
        val userId = "test-user-id"
        val initialUser = User(userId, "John", "john@example.com", "")
        val updatedUser = User(userId, "John Doe", "john.doe@example.com", "")
        
        every { userRepository.getUserById(userId) } returns initialUser
        every { userRepository.saveUser(any()) } just Runs
        
        composeTestRule.setContent {
            ProfileScreen(userId = userId)
        }
        
        // Navigate to edit mode
        composeTestRule
            .onNodeWithText("Edit Profile")
            .performClick()
        
        // Modify fields
        composeTestRule
            .onNodeWithText("Name")
            .performTextInput("John Doe")
        
        composeTestRule
            .onNodeWithText("Email")
            .performTextInput("john.doe@example.com")
        
        // Save
        composeTestRule
            .onNodeWithText("Save")
            .performClick()
        
        // Verify save was called
        verify(userRepository).saveUser(updatedUser)
    }
}
```

---

## Common Patterns and Anti-Patterns

### Good Patterns

#### 1. State Hoisting
```kotlin
// Good pattern
@Composable
fun UserCard(
    user: User,
    onUserClick: (User) -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.clickable { onUserClick(user) }
    ) {
        Text(user.name)
    }
}

// Usage
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(users) { user ->
            UserCard(
                user = user,
                onUserClick = { navigateToUserDetails(it.id) }
            )
        }
    }
}
```

#### 2. Separation of Concerns
```kotlin
// Good: Clean separation
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val updateUserUseCase: UpdateUserUseCase
) : ViewModel()

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> UserList(users = uiState.users)
        is UiState.Error -> ErrorScreen(uiState.message)
    }
}
```

### Anti-Patterns to Avoid

#### 1. ViewModel Overload
```kotlin
// Bad: Too much logic in ViewModel
@HiltViewModel
class BadViewModel @Inject constructor() : ViewModel() {
    fun formatUserName(user: User): String {
        // Too much business logic
        return "${user.firstName} ${user.lastName}"
    }
    
    fun calculateUserAge(birthDate: LocalDate): Int {
        // Age calculation logic
        return Period.between(birthDate, LocalDate.now()).years
    }
    
    fun validateEmail(email: String): Boolean {
        // Validation logic
        return email.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))
    }
}

// Better: Move business logic to use cases
class FormatUserNameUseCase @Inject constructor() {
    operator fun invoke(user: User): String {
        return "${user.firstName} ${user.lastName}"
    }
}
```

#### 2. Tight Coupling
```kotlin
// Bad: Direct dependency coupling
@HiltViewModel
class BadViewModel @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase,
    private val sharedPreferences: SharedPreferences
) : ViewModel() {
    // Too many direct dependencies
}

// Better: Use repository pattern
@HiltViewModel
class GoodViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    // Clean dependencies through abstractions
}
```

#### 3. Excessive Recomposition
```kotlin
// Bad: Causes unnecessary recomposition
@Composable
fun BadProfileCard(user: User) {
    // This will recompose on every state change
    Card {
        Text("User: ${user.name}") // Will recompose
        Text("Email: ${user.email}") // Will recompose
        Text("Last Login: ${DateTimeFormatter.ofPattern("yyyy-MM-dd").format(user.lastLogin)}") // Will recompose
    }
}

// Better: Optimize recomposition
@Composable
fun GoodProfileCard(user: User) {
    val formattedName by remember(user.name) { derivedStateOf { user.name } }
    val formattedEmail by remember(user.email) { derivedStateOf { user.email } }
    val formattedDate by remember(user.lastLogin) { 
        derivedStateOf { DateTimeFormatter.ofPattern("yyyy-MM-dd").format(user.lastLogin) } 
    }
    
    Card {
        Text("User: $formattedName")
        Text("Email: $formattedEmail")
        Text("Last Login: $formattedDate")
    }
}
```

---

## Migration Guide

### 1. From Traditional Views to Compose

#### Step-by-Step Migration

```kotlin
// Old approach with XML andfindViewById
class OldActivity : AppCompatActivity() {
    private lateinit var textView: TextView
    private lateinit var button: Button
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        textView = findViewById(R.id.textView)
        button = findViewById(R.id.button)
        
        button.setOnClickListener {
            textView.text = "Button clicked!"
        }
    }
}

// New approach with Compose
@AndroidEntryPoint
class NewActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            MyAppTheme {
                MainScreen()
            }
        }
    }
}

@Composable
fun MainScreen() {
    var text by remember { mutableStateOf("Hello World!") }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = text,
            style = MaterialTheme.typography.bodyLarge
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Button(
            onClick = { text = "Button clicked!" }
        ) {
            Text("Click me")
        }
    }
}
```

### 2. From Other DI Frameworks to Hilt

#### From Dagger to Hilt

```kotlin
// Old Dagger approach
@Component(modules = [NetworkModule::class, DatabaseModule::class])
interface AppComponent {
    fun inject(activity: MainActivity)
    fun inject(fragment: UserFragment)
}

@Module
abstract class NetworkModule {
    @Binds
    abstract fun bindApiService(apiServiceImpl: ApiServiceImpl): ApiService
}

// Hilt approach
@HiltAndroidApp
class MyApplication : Application()

@Module
@InstallIn(SingletonComponent::class)
abstract class NetworkModule {
    @Binds
    abstract fun bindApiService(apiServiceImpl: ApiServiceImpl): ApiService
}

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var apiService: ApiService
}
```

### 3. Hybrid Approach (Compose + Views)

```kotlin
@AndroidEntryPoint
class HybridActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            MyAppTheme {
                // Compose content
                Column {
                    Text("This is Compose content")
                    
                    // Android View embedded in Compose
                    AndroidView(
                        factory = { context ->
                            // Traditional Android View
                            TextView(context).apply {
                                text = "This is a traditional view"
                                textSize = 16f
                            }
                        }
                    )
                }
            }
        }
    }
}
```

---

## Resources and Further Reading

### Official Documentation

- [Hilt Documentation](https://dagger.dev/hilt/)
- [Jetpack Compose Documentation](https://developer.android.com/jetpack/compose)
- [Hilt and Compose Integration](https://developer.android.com/jetpack/compose/libraries#hilt)

### Sample Projects

- [Jetpack Compose Samples](https://github.com/android/compose-samples)
- [Jetpack Compose with Hilt](https://github.com/googlecodelabs/android-compose-codelabs)

### Best Practices and Guidelines

- [Android Architecture Guide](https://developer.android.com/topic/architecture)
- [Compose Performance Best Practices](https://developer.android.com/jetpack/compose/performance)
- [Dependency Injection Best Practices](https://dagger.dev/dev-guide/android)

### Tools and Libraries

- [Compose DevTools](https://developer.android.com/jetpack/compose/tools)
- [Compose Preview](https://developer.android.com/jetpack/compose/preview)
- [Hilt Compiler](https://dagger.dev/hilt-compiler)

### Community Resources

- [Android Developers YouTube](https://www.youtube.com/user/androiddevelopers)
- [Android Developers Blog](https://android-developers.googleblog.com/)
- [Kotlin Documentation](https://kotlinlang.org/docs/getting-started.html)

---

## Conclusion

Hilt and Jetpack Compose represent the modern approach to Android development, offering powerful tools for dependency injection and UI development respectively. When combined effectively, they enable developers to build robust, maintainable, and testable applications with significantly reduced boilerplate code.

Key takeaways:

1. **Hilt** simplifies dependency injection by providing Android-specific bindings and scopes
2. **Jetpack Compose** revolutionizes UI development with its declarative approach
3. **Integration** between both technologies is seamless with the `hiltViewModel()` function
4. **Architecture** patterns like Clean Architecture work well with both technologies
5. **Testing** becomes easier with proper separation of concerns and dependency injection

As Android development continues to evolve, these technologies will play a crucial role in building the next generation of Android applications. By mastering both Hilt and Jetpack Compose, developers can create modern, efficient, and maintainable Android apps that provide excellent user experiences.

Remember to always follow best practices, keep your modules focused, optimize for performance, and write comprehensive tests to ensure the quality of your applications.
