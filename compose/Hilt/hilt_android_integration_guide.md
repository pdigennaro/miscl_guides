# Comprehensive Guide to Hilt and Android Integration

*Author: MiniMax Agent*  
*Date: November 9, 2025*

## Table of Contents

1. [Introduction](#introduction)
2. [Hilt Fundamentals](#hilt-fundamentals)
3. [Setup and Installation](#setup-and-installation)
4. [Core Android Components](#core-android-components)
   - [Application Class](#application-class)
   - [Activities](#activities)
   - [Fragments](#fragments)
   - [Services](#services)
   - [Broadcast Receivers](#broadcast-receivers)
   - [ViewModels](#viewmodels)
5. [Android-Specific Bindings](#android-specific-bindings)
   - [Context Injection](#context-injection)
   - [Application Dependencies](#application-dependencies)
   - [Android Resources](#android-resources)
6. [Dependency Modules](#dependency-modules)
   - [Network Modules](#network-modules)
   - [Database Modules](#database-modules)
   - [SharedPreferences Modules](#sharedpreferences-modules)
   - [WorkManager Integration](#workmanager-integration)
7. [Scope Management](#scope-management)
8. [Advanced Patterns](#advanced-patterns)
9. [Testing with Hilt](#testing-with-hilt)
10. [Best Practices](#best-practices)
11. [Common Integration Scenarios](#common-integration-scenarios)
12. [Migration from Dagger](#migration-from-dagger)
13. [Troubleshooting](#troubleshooting)
14. [Resources and Further Reading](#resources-and-further-reading)

---

## Introduction

Hilt is Android's recommended dependency injection (DI) library that builds on top of Dagger, specifically designed for Android applications. This guide focuses on how Hilt integrates with Android's core components and provides seamless dependency injection throughout your Android application.

### Why Hilt for Android?

- **Android-Native**: Built specifically for Android with lifecycle-aware components
- **Reduced Boilerplate**: Automatic generation of Android components
- **Scope Management**: Automatic scope management tied to Android lifecycles
- **Context-Aware**: Built-in Android context and resource injection
- **WorkManager Integration**: Native support for WorkManager dependencies

---

## Hilt Fundamentals

### What Makes Hilt Android-Specific?

Unlike standard Dagger, Hilt provides:

1. **Automatic Android Component Generation**: Activities, Fragments, Services, etc.
2. **Lifecycle-Aware Scoping**: Components tied to Android component lifecycles
3. **Built-in Android Bindings**: Context, Application, Resources
4. **WorkManager Support**: Dependency injection for background workers
5. **ViewModel Integration**: Direct injection into Android ViewModels

### Key Differences from Dagger

```kotlin
// Dagger - Manual component setup
@Component(modules = [AppModule::class])
interface AppComponent {
    fun inject(activity: MainActivity)
    fun inject(fragment: UserFragment)
}

// Hilt - Automatic component management
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt automatically generates components
}

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var userRepository: UserRepository
}
```

---

## Setup and Installation

### 1. Add Dependencies

```gradle
// project-level build.gradle
plugins {
    id 'com.google.dagger.hilt.android' version '2.48' apply false
}

// app-level build.gradle
plugins {
    id 'com.google.dagger.hilt.android'
    id 'kotlin-kapt'
    id 'kotlin-parcelize' // Optional, for Parcelable support
}

android {
    // Enable Java 8+ features
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    
    kotlinOptions {
        jvmTarget = '17'
    }
}

dependencies {
    implementation 'com.google.dagger:hilt-android:2.48'
    implementation 'androidx.hilt:hilt-work:1.1.0'
    implementation 'androidx.hilt:hilt-navigation:1.1.0'
    
    // Annotation processors
    kapt 'com.google.dagger:hilt-compiler:2.48'
    kapt 'androidx.hilt:hilt-compiler:1.1.0'
}
```

### 2. Enable Hilt in Application

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt will:
    // 1. Generate Dagger components
    // 2. Create dependency graph
    // 3. Wire up component dependencies
}
```

---

## Core Android Components

### Application Class

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // Components available:
    // - SingletonComponent (Application scope)
    
    // Global dependencies are provided here:
    // - Network clients
    // - Database instances
    // - Shared preferences
    // - App-wide repositories
}
```

**Configuration Example:**
```kotlin
@HiltAndroidApp
class WeatherApp : Application() {
    
    override fun onCreate() {
        super.onCreate()
        
        // Application-wide initialization
        setupCrashReporting()
        initializeAnalytics()
    }
    
    private fun setupCrashReporting() {
        // Setup global error handling
    }
}
```

### Activities

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    // Activity-scoped dependencies
    @Inject
    lateinit var activityTracker: ActivityTracker
    
    // Singleton dependencies
    @Inject
    lateinit var userRepository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // Dependencies are available
        activityTracker.trackActivity(this)
    }
}

// Activity with ViewModel injection
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    
    // ViewModel with Hilt
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // ViewModel automatically injected
        viewModel.getUser("user123")
    }
}
```

### Fragments

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment() {
    
    // Fragment-scoped dependencies
    @Inject
    lateinit var fragmentHelper: FragmentHelper
    
    // ViewModel injection
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        // Dependencies available
        fragmentHelper.setupFragment(this)
        return inflater.inflate(R.layout.fragment_user, container, false)
    }
}
```

### Services

```kotlin
@AndroidEntryPoint
class DownloadService : Service() {
    
    @Inject
    lateinit var downloadManager: DownloadManager
    
    @Inject
    lateinit var notificationHelper: NotificationHelper
    
    override fun onCreate() {
        super.onCreate()
        // Service-scoped dependencies available
    }
    
    @Inject
    lateinit var analyticsTracker: AnalyticsTracker
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Track service start
        analyticsTracker.trackServiceStart("DownloadService")
        return super.onStartCommand(intent, flags, startId)
    }
}
```

### Broadcast Receivers

```kotlin
@AndroidEntryPoint
class NetworkReceiver : BroadcastReceiver() {
    
    @Inject
    lateinit var networkMonitor: NetworkMonitor
    
    override fun onReceive(context: Context, intent: Intent) {
        // Receiver-scoped dependencies available
        val isConnected = networkMonitor.isConnected()
        
        // Handle network change
        if (isConnected) {
            // Retry failed operations
        }
    }
}
```

### ViewModels

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val preferencesRepository: PreferencesRepository
) : ViewModel() {
    
    private val _userState = MutableLiveData<User>()
    val userState: LiveData<User> = _userState
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val user = userRepository.getUser(userId)
                _userState.value = user
            } catch (e: Exception) {
                // Handle error
            }
        }
    }
    
    fun updateUserPreferences(preferences: UserPreferences) {
        viewModelScope.launch {
            preferencesRepository.updatePreferences(userId, preferences)
        }
    }
}
```

**Usage in Activity/Fragment:**
```kotlin
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        viewModel.userState.observe(this) { user ->
            // Update UI with user data
        }
        
        viewModel.loadUser("user123")
    }
}
```

---

## Android-Specific Bindings

### Context Injection

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ContextModule {
    
    @Provides
    @ApplicationContext
    fun provideApplicationContext(application: Application): Context {
        return application.applicationContext
    }
    
    @Provides
    fun provideActivityContext(activity: Activity): Context {
        return activity
    }
}

// Usage
@AndroidEntryPoint
class MyActivity : AppCompatActivity() {
    
    @Inject
    @ApplicationContext
    lateinit var appContext: Context
    
    @Inject
    lateinit var activityContext: Context
}
```

### Application Dependencies

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ApplicationModule {
    
    @Provides
    fun provideSharedPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }
    
    @Provides
    fun provideDatabase(
        @ApplicationContext context: Context
    ): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
    
    @Provides
    fun provideAssetManager(
        @ApplicationContext context: Context
    ): AssetManager {
        return context.assets
    }
}
```

### Android Resources

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ResourcesModule {
    
    @Provides
    fun provideResources(
        @ApplicationContext context: Context
    ): Resources {
        return context.resources
    }
    
    @Provides
    fun provideStringResolver(
        @ApplicationContext context: Context
    ): StringResolver {
        return StringResolver { stringId ->
            context.getString(stringId)
        }
    }
}

class StringResolver @Inject constructor(
    private val getString: (Int) -> String
) {
    fun getString(stringId: Int): String = getString(stringId)
}
```

---

## Dependency Modules

### Network Modules

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        val loggingInterceptor = HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }
        
        return OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)
            .addInterceptor { chain ->
                val originalRequest = chain.request()
                val requestWithToken = originalRequest.newBuilder()
                    .header("Authorization", "Bearer ${getAuthToken()}")
                    .build()
                chain.proceed(requestWithToken)
            }
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .build()
    }
    
    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
    
    private fun getAuthToken(): String {
        // Retrieve token from secure storage
        return ""
    }
}
```

### Database Modules

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context
    ): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        )
        .addMigrations(MIGRATION_1_2)
        .addCallback(object : RoomDatabase.Callback() {
            override fun onCreate(db: SupportSQLiteDatabase) {
                super.onCreate(db)
                // Seed initial data
            }
        })
        .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
    
    @Provides
    fun provideProductDao(database: AppDatabase): ProductDao {
        return database.productDao()
    }
    
    @Provides
    fun provideRepository(
        userDao: UserDao,
        productDao: ProductDao,
        apiService: ApiService
    ): Repository {
        return RepositoryImpl(userDao, productDao, apiService)
    }
}

// Migration example
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE users ADD COLUMN phone TEXT"
        )
    }
}
```

### SharedPreferences Modules

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object PreferencesModule {
    
    @Provides
    @Named("auth")
    fun provideAuthPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return context.getSharedPreferences("auth_prefs", Context.MODE_PRIVATE)
    }
    
    @Provides
    @Named("settings")
    fun provideSettingsPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return context.getSharedPreferences("settings_prefs", Context.MODE_PRIVATE)
    }
    
    @Provides
    fun providePreferencesRepository(
        @Named("auth") authPreferences: SharedPreferences,
        @Named("settings") settingsPreferences: SharedPreferences
    ): PreferencesRepository {
        return PreferencesRepositoryImpl(authPreferences, settingsPreferences)
    }
}

@Singleton
class PreferencesRepositoryImpl @Inject constructor(
    private val authPreferences: SharedPreferences,
    private val settingsPreferences: SharedPreferences
) : PreferencesRepository {
    
    override fun getAuthToken(): String? {
        return authPreferences.getString(KEY_AUTH_TOKEN, null)
    }
    
    override fun setAuthToken(token: String) {
        authPreferences.edit()
            .putString(KEY_AUTH_TOKEN, token)
            .apply()
    }
    
    override fun isDarkModeEnabled(): Boolean {
        return settingsPreferences.getBoolean(KEY_DARK_MODE, false)
    }
    
    override fun setDarkModeEnabled(enabled: Boolean) {
        settingsPreferences.edit()
            .putBoolean(KEY_DARK_MODE, enabled)
            .apply()
    }
    
    companion object {
        private const val KEY_AUTH_TOKEN = "auth_token"
        private const val KEY_DARK_MODE = "dark_mode"
    }
}
```

### WorkManager Integration

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val syncRepository: SyncRepository
) : CoroutineWorker(context, workerParams) {
    
    override suspend fun doWork(): Result {
        return try {
            syncRepository.performFullSync()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

@Module
@InstallIn(WorkerComponent::class)
object WorkerModule {
    
    @Provides
    @Singleton
    fun provideSyncRepository(
        apiService: ApiService,
        database: AppDatabase
    ): SyncRepository {
        return SyncRepositoryImpl(apiService, database)
    }
}

// Enqueue work with Hilt
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        enqueuePeriodicSync()
    }
    
    private fun enqueuePeriodicSync() {
        val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
        
        WorkManager.getInstance(this)
            .enqueueUniquePeriodicWork(
                "sync_work",
                ExistingPeriodicWorkPolicy.REPLACE,
                syncRequest
            )
    }
}
```

---

## Scope Management

### Android Component Scopes

```kotlin
@Singleton
// Application scope - lives for entire app lifetime
class AppDatabase @Inject constructor(
    @ApplicationContext context: Context
) {
    // Singleton implementation
}

@ActivityScoped
// Activity scope - recreated with each activity
class ActivityTracker @Inject constructor(
    private val activity: Activity
) {
    fun trackActivity() {
        // Track activity lifecycle
    }
}

@FragmentScoped
// Fragment scope - recreated with each fragment
class FragmentHelper @Inject constructor(
    private val fragment: Fragment
) {
    fun setupFragment() {
        // Setup fragment-specific functionality
    }
}

@ServiceScoped
// Service scope - lives with service instance
class DownloadManager @Inject constructor() {
    // Service-specific implementation
}
```

### Custom Scopes

```kotlin
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class UserScoped

@UserScoped
@Component
interface UserComponent {
    fun inject(userActivity: UserActivity)
}

@UserScoped
class UserManager @Inject constructor() {
    // User-specific manager
}
```

---

## Advanced Patterns

### Multi-Module Support

```kotlin
// Core module
@Module
@InstallIn(SingletonComponent::class)
abstract class CoreModule {
    
    @Binds
    abstract fun bindLogger(loggerImpl: LoggerImpl): Logger
    
    companion object {
        @Provides
        fun provideNetworkInterceptor(): Interceptor {
            return NetworkInterceptor()
        }
    }
}

// Feature module
@Module
@InstallIn(ActivityComponent::class)
abstract class UserModule {
    
    @Binds
    abstract fun bindUserRepository(userRepositoryImpl: UserRepositoryImpl): UserRepository
}

// App module
@Module(includes = [CoreModule::class, NetworkModule::class])
@InstallIn(SingletonComponent::class)
object AppModule
```

### Component Dependencies

```kotlin
// Feature-specific component with dependencies
@ActivityScoped
@Component(
    dependencies = [ApplicationComponent::class],
    modules = [UserModule::class]
)
interface UserComponent {
    fun inject(activity: UserActivity)
    
    @Component.Builder
    interface Builder {
        fun applicationComponent(appComponent: ApplicationComponent): Builder
        fun build(): UserComponent
    }
}
```

### Qualifier Annotations

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthRetrofit

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class PublicRetrofit

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @AuthRetrofit
    fun provideAuthRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://auth.example.com/")
            .build()
    }
    
    @Provides
    @PublicRetrofit
    fun providePublicRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .build()
    }
}

// Usage
@AndroidEntryPoint
class AuthActivity : AppCompatActivity() {
    
    @Inject
    @AuthRetrofit
    lateinit var authRetrofit: Retrofit
    
    @Inject
    @PublicRetrofit
    lateinit var publicRetrofit: Retrofit
}
```

---

## Testing with Hilt

### Unit Testing

```kotlin
@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class UserRepositoryTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @Inject
    @field:Mock
    lateinit var apiService: ApiService
    
    @Inject
    @field:Mock
    lateinit var database: AppDatabase
    
    @Inject
    lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testUserRepository() = runTest {
        // Given
        val mockUser = User("1", "John", "john@example.com")
        every { apiService.getUser("1") } returns mockUser
        
        // When
        val result = userRepository.getUser("1")
        
        // Then
        assertEquals(mockUser, result)
    }
}
```

### Instrumentation Testing

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class MainActivityTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val activityTestRule = ActivityTestRule(MainActivity::class.java)
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testActivityLaunch() {
        // Test activity behavior
        onView(withId(R.id.recyclerView))
            .check(matches(isDisplayed()))
    }
}
```

### Test Modules

```kotlin
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
            .addConverterFactory(MoshiConverterFactory.create())
            .build()
    }
    
    @Provides
    fun provideMockApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

---

## Best Practices

### 1. Module Organization

```kotlin
// Good: Organize by feature and responsibility
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }
    
    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

### 2. Constructor Injection

```kotlin
// Good: Use constructor injection
class UserRepository @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase,
    private val preferences: SharedPreferences
) {
    // Implementation
}

// Good: Interface binding
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindUserRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
}
```

### 3. Context Usage

```kotlin
// Good: Use @ApplicationContext for application-wide operations
@AndroidEntryPoint
class AnalyticsManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun trackEvent(event: String) {
        // Application context is safe for long-lived operations
    }
}

// Good: Use Activity context when needed
@AndroidEntryPoint
class ActivityNavigator @Inject constructor(
    private val activity: Activity
) {
    fun navigateToNextScreen() {
        // Activity context for navigation
        val intent = Intent(activity, NextActivity::class.java)
        activity.startActivity(intent)
    }
}
```

### 4. Error Handling

```kotlin
@AndroidEntryPoint
class ErrorHandler @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun handleError(exception: Exception) {
        when (exception) {
            is NetworkException -> showNetworkError()
            is AuthException -> redirectToLogin()
            is ValidationException -> showValidationError(exception.message)
            else -> showGenericError()
        }
    }
    
    private fun showNetworkError() {
        Toast.makeText(context, "Network error occurred", Toast.LENGTH_SHORT).show()
    }
    
    private fun showGenericError() {
        // Show generic error message
    }
}
```

---

## Common Integration Scenarios

### 1. Login Flow

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val preferencesRepository: PreferencesRepository
) : ViewModel() {
    
    private val _loginState = MutableLiveData<LoginState>()
    val loginState: LiveData<LoginState> = _loginState
    
    fun login(email: String, password: String) {
        _loginState.value = LoginState.Loading
        
        viewModelScope.launch {
            try {
                val result = authRepository.login(email, password)
                preferencesRepository.setAuthToken(result.token)
                _loginState.value = LoginState.Success(result)
            } catch (e: Exception) {
                _loginState.value = LoginState.Error(e.message ?: "Login failed")
            }
        }
    }
}

@AndroidEntryPoint
class LoginActivity : AppCompatActivity() {
    
    private val viewModel: LoginViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
        
        viewModel.loginState.observe(this) { state ->
            when (state) {
                is LoginState.Loading -> showLoading()
                is LoginState.Success -> navigateToMain()
                is LoginState.Error -> showError(state.message)
            }
        }
    }
}
```

### 2. Repository Pattern

```kotlin
// Repository interface
interface UserRepository {
    suspend fun getUser(id: String): User
    suspend fun getAllUsers(): List<User>
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
}

// Repository implementation
class UserRepositoryImpl @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase,
    private val cacheManager: CacheManager
) : UserRepository {
    
    override suspend fun getUser(id: String): User {
        return try {
            // Try network first
            val user = apiService.getUser(id)
            cacheManager.cacheUser(user)
            database.userDao().insert(user)
            user
        } catch (e: Exception) {
            // Fall back to database
            database.userDao().getUserById(id) ?: throw NoUserFoundException()
        }
    }
    
    override suspend fun saveUser(user: User) {
        database.userDao().insert(user)
        try {
            apiService.saveUser(user)
        } catch (e: Exception) {
            // Handle offline scenario
            cacheManager.markUserAsPendingSync(user.id)
        }
    }
}

// Module setup
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindUserRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
}
```

### 3. Background Tasks

```kotlin
@HiltWorker
class DataSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val syncRepository: SyncRepository,
    private val networkMonitor: NetworkMonitor
) : CoroutineWorker(context, workerParams) {
    
    override suspend fun doWork(): Result {
        return if (networkMonitor.isConnected()) {
            try {
                syncRepository.syncAllData()
                Result.success()
            } catch (e: Exception) {
                Result.retry()
            }
        } else {
            Result.failure()
        }
    }
}

@AndroidEntryPoint
class SyncManager @Inject constructor(
    private val workManager: WorkManager,
    private val preferencesRepository: PreferencesRepository
) {
    
    fun schedulePeriodicSync() {
        val syncRequest = PeriodicWorkRequestBuilder<DataSyncWorker>(
            1, TimeUnit.HOURS
        ).setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        ).build()
        
        workManager.enqueueUniquePeriodicWork(
            "data_sync",
            ExistingPeriodicWorkPolicy.REPLACE,
            syncRequest
        )
    }
    
    fun triggerImmediateSync() {
        val syncRequest = OneTimeWorkRequestBuilder<DataSyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            ).build()
        
        workManager.enqueueUniqueWork(
            "immediate_sync",
            ExistingWorkPolicy.REPLACE,
            syncRequest
        )
    }
}
```

---

## Migration from Dagger

### Step-by-Step Migration

#### 1. Update Application Class

```kotlin
// Before (Dagger)
class MyApplication : Application() {
    lateinit var appComponent: AppComponent
    
    override fun onCreate() {
        super.onCreate()
        appComponent = DaggerAppComponent.builder()
            .applicationModule(ApplicationModule(this))
            .build()
        appComponent.inject(this)
    }
}

// After (Hilt)
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt automatically generates components
}
```

#### 2. Update Activities

```kotlin
// Before (Dagger)
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var userRepository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        (application as MyApplication).appComponent.inject(this)
    }
}

// After (Hilt)
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var userRepository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Dependencies automatically available
    }
}
```

#### 3. Update Modules

```kotlin
// Before (Dagger)
@Module
@Component(modules = [NetworkModule::class])
class AppComponent {
    @Component.Builder
    interface Builder {
        fun build(): AppComponent
    }
}

// After (Hilt)
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

#### 4. Update ViewModels

```kotlin
// Before (Dagger)
class UserViewModel : ViewModel() {
    @Inject
    lateinit var userRepository: UserRepository
    
    constructor(userRepository: UserRepository) {
        this.userRepository = userRepository
    }
}

// After (Hilt)
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    // Implementation
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. "Hilt is not configured" Error

```gradle
// Add to project build.gradle
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.48'
    }
}

// Add to app build.gradle
apply plugin: 'dagger.hilt.android.plugin'
apply plugin: 'kotlin-kapt'
```

#### 2. "Missing @AndroidEntryPoint" Error

```kotlin
// Error: Activities must be annotated with @AndroidEntryPoint
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    // All activities, fragments, services, etc. need this annotation
}
```

#### 3. "Could not be provided" Error

```kotlin
// Problem: Missing module or incorrect scope
@Module
@InstallIn(ActivityComponent::class) // Wrong scope for singleton dependency
object BadModule {
    @Provides
    fun provideDatabase(): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "db")
            .build()
    }
}

// Solution: Use correct scope
@Module
@InstallIn(SingletonComponent::class) // Correct scope
object GoodModule {
    @Provides
    @Singleton // Add scope annotation
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "db")
            .build()
    }
}
```

#### 4. Context Injection Issues

```kotlin
// Problem: Using Activity context where Application context is needed
class SingletonManager @Inject constructor(
    private val context: Context // Should specify scope
) {
    fun initialize() {
        // This will fail if context is Activity-scoped
    }
}

// Solution: Specify context type
class SingletonManager @Inject constructor(
    @ApplicationContext private val context: Context // Correct
) {
    fun initialize() {
        // Safe to use across activity lifecycle
    }
}
```

---

## Resources and Further Reading

### Official Documentation

- [Hilt Documentation](https://dagger.dev/hilt/)
- [Android Developer Hilt Guide](https://developer.android.com/training/dependency-injection/hilt-android)
- [Hilt and Dagger APIs](https://dagger.dev/dev-guide/android)

### Sample Projects

- [Hilt Samples](https://github.com/googlecodelabs/android-hilt)
- [Android Architecture Samples](https://github.com/android/architecture-samples)

### Tools and Utilities

- [Hilt Inspector](https://dagger.dev/dev-guide/android#inspecting-your-graph) - Visualize dependency graphs
- [Dagger Visualizer](https://dagger.dev/dev-guide/graph-visualization) - Component visualization

### Community Resources

- [Android Developers Blog](https://android-developers.googleblog.com/)
- [Google I/O Sessions on DI](https://io.google/2020/)

---

## Conclusion

Hilt provides a robust, Android-native dependency injection solution that seamlessly integrates with Android's component lifecycle. Key benefits include:

1. **Reduced Boilerplate**: Automatic component generation and wiring
2. **Android-Native**: Built specifically for Android with lifecycle awareness
3. **Type Safety**: Compile-time dependency graph validation
4. **Testing Support**: Built-in testing utilities and mock capabilities
5. **Scope Management**: Automatic lifecycle-bound scope management

### Key Takeaways:

- **Always annotate** Android components with `@AndroidEntryPoint`
- **Use appropriate scopes** (`@Singleton`, `@ActivityScoped`, etc.)
- **Prefer constructor injection** over field injection
- **Organize modules** by feature and responsibility
- **Test with** Hilt's testing utilities for reliable unit and integration tests

By following these patterns and best practices, you can build maintainable, testable Android applications with robust dependency injection using Hilt.
