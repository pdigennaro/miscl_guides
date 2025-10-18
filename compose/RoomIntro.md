# Complete Guide to Android Room

## What is Room?

Room is an abstraction layer over SQLite that provides a more robust database access mechanism while harnessing the full power of SQLite. It's part of Android Jetpack and simplifies database operations with compile-time verification of SQL queries and observable database queries.

## Why Use Room?

- **Compile-time verification** of SQL queries
- **Reduced boilerplate code** compared to raw SQLite
- **Built-in migration support** for database schema changes
- **Integration with LiveData, Flow, and RxJava** for reactive programming
- **Type converters** for complex data types
- **Better testability** with in-memory database support

## Core Components

Room has three main components:

1. **Entity**: Represents a table in the database
2. **DAO (Data Access Object)**: Contains methods for database operations
3. **Database**: Holds the database and serves as the main access point

## Setup

Add Room dependencies to your `build.gradle` file:

```gradle
dependencies {
    def room_version = "2.6.1"

    implementation "androidx.room:room-runtime:$room_version"
    kapt "androidx.room:room-compiler:$room_version" // For Kotlin
    // annotationProcessor "androidx.room:room-compiler:$room_version" // For Java

    // Optional - Kotlin Extensions and Coroutines support
    implementation "androidx.room:room-ktx:$room_version"

    // Optional - RxJava support
    implementation "androidx.room:room-rxjava2:$room_version"

    // Optional - Test helpers
    testImplementation "androidx.room:room-testing:$room_version"
}
```

## Creating an Entity

An Entity represents a table in your database:

```kotlin
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "first_name")
    val firstName: String,
    
    @ColumnInfo(name = "last_name")
    val lastName: String,
    
    val email: String,
    
    val age: Int
)
```

### Entity Annotations

- `@Entity`: Marks a class as a database table
- `@PrimaryKey`: Designates the primary key (use `autoGenerate = true` for auto-increment)
- `@ColumnInfo`: Customizes column name and properties
- `@Ignore`: Excludes a field from the database
- `@Embedded`: Nests fields from another class

## Creating a DAO

The DAO defines methods for accessing the database:

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Long): User?
    
    @Query("SELECT * FROM users WHERE age > :minAge")
    fun getUsersOlderThan(minAge: Int): LiveData<List<User>>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User): Long
    
    @Insert
    suspend fun insertUsers(vararg users: User)
    
    @Update
    suspend fun updateUser(user: User)
    
    @Delete
    suspend fun deleteUser(user: User)
    
    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteUserById(userId: Long)
    
    @Query("DELETE FROM users")
    suspend fun deleteAllUsers()
}
```

### DAO Annotations

- `@Query`: For custom SQL queries
- `@Insert`: Inserts one or more entities
- `@Update`: Updates existing entities
- `@Delete`: Deletes entities
- `@Transaction`: Marks a method as transactional

### Conflict Strategies

- `OnConflictStrategy.REPLACE`: Replace existing data
- `OnConflictStrategy.IGNORE`: Ignore conflicts
- `OnConflictStrategy.ABORT`: Abort transaction (default)

## Creating the Database

```kotlin
@Database(entities = [User::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

## Relationships

### One-to-Many Relationship

```kotlin
@Entity
data class School(
    @PrimaryKey val schoolId: Long,
    val name: String
)

@Entity(
    foreignKeys = [ForeignKey(
        entity = School::class,
        parentColumns = ["schoolId"],
        childColumns = ["schoolId"],
        onDelete = ForeignKey.CASCADE
    )]
)
data class Student(
    @PrimaryKey val studentId: Long,
    val name: String,
    val schoolId: Long
)

data class SchoolWithStudents(
    @Embedded val school: School,
    @Relation(
        parentColumn = "schoolId",
        entityColumn = "schoolId"
    )
    val students: List<Student>
)

@Dao
interface SchoolDao {
    @Transaction
    @Query("SELECT * FROM School")
    fun getSchoolsWithStudents(): List<SchoolWithStudents>
}
```

### Many-to-Many Relationship

```kotlin
@Entity
data class Student(
    @PrimaryKey val studentId: Long,
    val name: String
)

@Entity
data class Course(
    @PrimaryKey val courseId: Long,
    val name: String
)

@Entity(primaryKeys = ["studentId", "courseId"])
data class StudentCourseCrossRef(
    val studentId: Long,
    val courseId: Long
)

data class StudentWithCourses(
    @Embedded val student: Student,
    @Relation(
        parentColumn = "studentId",
        entityColumn = "courseId",
        associateBy = Junction(StudentCourseCrossRef::class)
    )
    val courses: List<Course>
)
```

## Type Converters

For storing complex data types:

```kotlin
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? {
        return value?.let { Date(it) }
    }
    
    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? {
        return date?.time
    }
    
    @TypeConverter
    fun fromStringList(value: String): List<String> {
        return value.split(",").map { it.trim() }
    }
    
    @TypeConverter
    fun toStringList(list: List<String>): String {
        return list.joinToString(",")
    }
}

@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

## Database Migrations

When you change your database schema, you need to provide a migration:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN phone_number TEXT")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("CREATE TABLE IF NOT EXISTS addresses (id INTEGER PRIMARY KEY NOT NULL, street TEXT NOT NULL)")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

### Fallback Strategy

For development, you can use destructive migration:

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .fallbackToDestructiveMigration() // Deletes and recreates database
    .build()
```

## LiveData Integration - Deep Dive

LiveData is an observable data holder that's lifecycle-aware, making it perfect for Room database operations. Room has built-in support for returning LiveData from DAO queries.

### Basic LiveData Setup

```kotlin
@Dao
interface UserDao {
    // Room automatically handles LiveData updates when data changes
    @Query("SELECT * FROM users")
    fun getAllUsers(): LiveData<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserById(userId: Long): LiveData<User?>
    
    @Query("SELECT * FROM users WHERE age BETWEEN :minAge AND :maxAge")
    fun getUsersByAgeRange(minAge: Int, maxAge: Int): LiveData<List<User>>
    
    // Still use suspend functions for write operations
    @Insert
    suspend fun insertUser(user: User): Long
    
    @Update
    suspend fun updateUser(user: User)
    
    @Delete
    suspend fun deleteUser(user: User)
}
```

### Repository Pattern with LiveData

Create a repository to abstract the data source:

```kotlin
class UserRepository(private val userDao: UserDao) {
    
    // LiveData - automatically updates when database changes
    val allUsers: LiveData<List<User>> = userDao.getAllUsers()
    
    // Function to get users by age range
    fun getUsersByAge(minAge: Int, maxAge: Int): LiveData<List<User>> {
        return userDao.getUsersByAgeRange(minAge, maxAge)
    }
    
    // Function to get single user
    fun getUserById(userId: Long): LiveData<User?> {
        return userDao.getUserById(userId)
    }
    
    // Suspend functions for write operations
    suspend fun insert(user: User): Long {
        return userDao.insertUser(user)
    }
    
    suspend fun update(user: User) {
        userDao.updateUser(user)
    }
    
    suspend fun delete(user: User) {
        userDao.deleteUser(user)
    }
    
    suspend fun deleteAll() {
        userDao.deleteAllUsers()
    }
}
```

### ViewModel with LiveData

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    
    // Expose LiveData from repository
    val allUsers: LiveData<List<User>> = repository.allUsers
    
    // LiveData that can be updated based on parameters
    private val _selectedUserId = MutableLiveData<Long>()
    val selectedUser: LiveData<User?> = _selectedUserId.switchMap { userId ->
        repository.getUserById(userId)
    }
    
    // Filtered users with Transformations
    val adultUsers: LiveData<List<User>> = allUsers.map { users ->
        users.filter { it.age >= 18 }
    }
    
    // Combined LiveData using MediatorLiveData
    private val _minAge = MutableLiveData<Int>()
    private val _maxAge = MutableLiveData<Int>()
    
    val filteredUsers = MediatorLiveData<List<User>>().apply {
        var currentMin = 0
        var currentMax = 100
        
        addSource(_minAge) { min ->
            currentMin = min
            value = allUsers.value?.filter { 
                it.age >= currentMin && it.age <= currentMax 
            } ?: emptyList()
        }
        
        addSource(_maxAge) { max ->
            currentMax = max
            value = allUsers.value?.filter { 
                it.age >= currentMin && it.age <= currentMax 
            } ?: emptyList()
        }
        
        addSource(allUsers) { users ->
            value = users.filter { 
                it.age >= currentMin && it.age <= currentMax 
            }
        }
    }
    
    // Functions to trigger operations
    fun selectUser(userId: Long) {
        _selectedUserId.value = userId
    }
    
    fun setAgeRange(min: Int, max: Int) {
        _minAge.value = min
        _maxAge.value = max
    }
    
    fun insertUser(user: User) = viewModelScope.launch {
        repository.insert(user)
    }
    
    fun updateUser(user: User) = viewModelScope.launch {
        repository.update(user)
    }
    
    fun deleteUser(user: User) = viewModelScope.launch {
        repository.delete(user)
    }
}
```

### Using LiveData in Activities/Fragments

```kotlin
class UserListFragment : Fragment() {
    
    private val viewModel: UserViewModel by viewModels()
    private lateinit var adapter: UserAdapter
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        setupRecyclerView()
        observeUsers()
    }
    
    private fun setupRecyclerView() {
        adapter = UserAdapter(
            onUserClick = { user -> viewModel.selectUser(user.id) },
            onDeleteClick = { user -> viewModel.deleteUser(user) }
        )
        binding.recyclerView.adapter = adapter
    }
    
    private fun observeUsers() {
        // Observe all users
        viewModel.allUsers.observe(viewLifecycleOwner) { users ->
            adapter.submitList(users)
            binding.emptyView.isVisible = users.isEmpty()
        }
        
        // Observe selected user
        viewModel.selectedUser.observe(viewLifecycleOwner) { user ->
            user?.let { showUserDetails(it) }
        }
        
        // Observe filtered users
        viewModel.adultUsers.observe(viewLifecycleOwner) { adults ->
            updateAdultCount(adults.size)
        }
    }
    
    private fun showUserDetails(user: User) {
        // Navigate to details or show dialog
    }
    
    private fun updateAdultCount(count: Int) {
        binding.adultCountText.text = "Adults: $count"
    }
}
```

### LiveData Transformations

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    
    val allUsers: LiveData<List<User>> = repository.allUsers
    
    // map: Transform each emission
    val userNames: LiveData<List<String>> = allUsers.map { users ->
        users.map { it.firstName + " " + it.lastName }
    }
    
    // switchMap: Switch to different LiveData based on input
    private val _searchQuery = MutableLiveData<String>()
    val searchResults: LiveData<List<User>> = _searchQuery.switchMap { query ->
        if (query.isNullOrBlank()) {
            repository.allUsers
        } else {
            repository.searchUsers(query)
        }
    }
    
    // distinctUntilChanged: Only emit when value changes
    val userCount: LiveData<Int> = allUsers.map { it.size }.distinctUntilChanged()
    
    fun search(query: String) {
        _searchQuery.value = query
    }
}
```

### Complex LiveData Scenarios

```kotlin
class DashboardViewModel(
    private val userRepository: UserRepository,
    private val orderRepository: OrderRepository
) : ViewModel() {
    
    // Combine multiple LiveData sources
    val dashboardData: LiveData<DashboardData> = MediatorLiveData<DashboardData>().apply {
        var users: List<User>? = null
        var orders: List<Order>? = null
        
        fun update() {
            val currentUsers = users
            val currentOrders = orders
            if (currentUsers != null && currentOrders != null) {
                value = DashboardData(
                    totalUsers = currentUsers.size,
                    activeUsers = currentUsers.count { it.isActive },
                    totalOrders = currentOrders.size,
                    recentOrders = currentOrders.take(5)
                )
            }
        }
        
        addSource(userRepository.allUsers) { userList ->
            users = userList
            update()
        }
        
        addSource(orderRepository.allOrders) { orderList ->
            orders = orderList
            update()
        }
    }
}

data class DashboardData(
    val totalUsers: Int,
    val activeUsers: Int,
    val totalOrders: Int,
    val recentOrders: List<Order>
)
```

## Flow vs LiveData

While this guide focuses on LiveData, here's a quick comparison:

### Using Flow (Modern Approach)

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
}

// In ViewModel
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    val users: StateFlow<List<User>> = repository.allUsers
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}

// In Fragment/Activity
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.users.collect { users ->
            adapter.submitList(users)
        }
    }
}
```

**When to use LiveData:**
- You're comfortable with LiveData
- Integration with existing LiveData-based code
- Simpler lifecycle handling in UI layer

**When to use Flow:**
- New projects
- Want more powerful operators
- Better testability
- Kotlin-first approach

## Hilt Integration - Complete Guide

Hilt is Android's recommended dependency injection library. Here's how to integrate it with Room for clean architecture.

### Hilt Setup

Add Hilt dependencies to your `build.gradle`:

```gradle
plugins {
    id 'com.google.dagger.hilt.android'
    id 'kotlin-kapt'
}

dependencies {
    def hilt_version = "2.48"
    
    implementation "com.google.dagger:hilt-android:$hilt_version"
    kapt "com.google.dagger:hilt-compiler:$hilt_version"
    
    // For ViewModel integration
    implementation "androidx.hilt:hilt-navigation-compose:1.1.0" // If using Compose
    // or
    implementation "androidx.fragment:fragment-ktx:1.6.2" // For traditional ViewModels
}
```

Project-level `build.gradle`:

```gradle
plugins {
    id 'com.google.dagger.hilt.android' version '2.48' apply false
}
```

### Application Class

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

Update `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    ...>
</application>
```

### Database Module

Create a module to provide Room database and DAOs:

```kotlin
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
        )
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
            .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
    
    @Provides
    fun provideOrderDao(database: AppDatabase): OrderDao {
        return database.orderDao()
    }
    
    @Provides
    fun provideProductDao(database: AppDatabase): ProductDao {
        return database.productDao()
    }
}
```

### Repository with Constructor Injection

```kotlin
class UserRepository @Inject constructor(
    private val userDao: UserDao
) {
    val allUsers: LiveData<List<User>> = userDao.getAllUsers()
    
    fun getUserById(userId: Long): LiveData<User?> {
        return userDao.getUserById(userId)
    }
    
    suspend fun insert(user: User): Long {
        return userDao.insertUser(user)
    }
    
    suspend fun update(user: User) {
        userDao.updateUser(user)
    }
    
    suspend fun delete(user: User) {
        userDao.deleteUser(user)
    }
    
    suspend fun searchUsers(query: String): List<User> {
        return userDao.searchUsers("%$query%")
    }
}
```

### ViewModel with Hilt

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    val allUsers: LiveData<List<User>> = repository.allUsers
    
    private val _selectedUserId = MutableLiveData<Long>()
    val selectedUser: LiveData<User?> = _selectedUserId.switchMap { userId ->
        repository.getUserById(userId)
    }
    
    fun selectUser(userId: Long) {
        _selectedUserId.value = userId
    }
    
    fun insertUser(user: User) = viewModelScope.launch {
        repository.insert(user)
    }
    
    fun updateUser(user: User) = viewModelScope.launch {
        repository.update(user)
    }
    
    fun deleteUser(user: User) = viewModelScope.launch {
        repository.delete(user)
    }
}
```

### Fragment/Activity with Hilt

```kotlin
@AndroidEntryPoint
class UserListFragment : Fragment() {
    
    private val viewModel: UserViewModel by viewModels()
    private lateinit var binding: FragmentUserListBinding
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewModel.allUsers.observe(viewLifecycleOwner) { users ->
            // Update UI
        }
    }
}

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Setup
    }
}
```

### Multiple Databases with Hilt

If you need multiple databases:

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class UserDatabase

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AnalyticsDatabase

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    @UserDatabase
    fun provideUserDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "user_database"
        ).build()
    }
    
    @Provides
    @Singleton
    @AnalyticsDatabase
    fun provideAnalyticsDatabase(@ApplicationContext context: Context): AnalyticsDatabase {
        return Room.databaseBuilder(
            context,
            AnalyticsDatabase::class.java,
            "analytics_database"
        ).build()
    }
    
    @Provides
    fun provideUserDao(@UserDatabase database: AppDatabase): UserDao {
        return database.userDao()
    }
    
    @Provides
    fun provideEventDao(@AnalyticsDatabase database: AnalyticsDatabase): EventDao {
        return database.eventDao()
    }
}

// Usage in Repository
class UserRepository @Inject constructor(
    @UserDatabase private val userDao: UserDao
) {
    // Implementation
}

class AnalyticsRepository @Inject constructor(
    @AnalyticsDatabase private val eventDao: EventDao
) {
    // Implementation
}
```

### Use Cases / Interactors Pattern with Hilt

For more complex business logic:

```kotlin
class GetUserByIdUseCase @Inject constructor(
    private val repository: UserRepository
) {
    operator fun invoke(userId: Long): LiveData<User?> {
        return repository.getUserById(userId)
    }
}

class InsertUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(user: User): Long {
        // Add validation or business logic here
        if (user.email.isBlank()) {
            throw IllegalArgumentException("Email cannot be empty")
        }
        return repository.insert(user)
    }
}

@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUserByIdUseCase: GetUserByIdUseCase,
    private val insertUserUseCase: InsertUserUseCase
) : ViewModel() {
    
    private val _selectedUserId = MutableLiveData<Long>()
    val selectedUser: LiveData<User?> = _selectedUserId.switchMap { userId ->
        getUserByIdUseCase(userId)
    }
    
    fun insertUser(user: User) = viewModelScope.launch {
        try {
            insertUserUseCase(user)
        } catch (e: IllegalArgumentException) {
            // Handle error
        }
    }
}
```

### Scoped Dependencies

Different scopes for different use cases:

```kotlin
// Singleton - single instance throughout app lifetime
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        // ...
    }
}

// ActivityRetainedComponent - survives configuration changes
@Module
@InstallIn(ActivityRetainedComponent::class)
object SessionModule {
    @Provides
    fun provideSessionManager(repository: UserRepository): SessionManager {
        return SessionManager(repository)
    }
}

// ActivityComponent - tied to activity lifecycle
@Module
@InstallIn(ActivityComponent::class)
object ActivityModule {
    @Provides
    fun provideActivitySpecificHelper(): ActivityHelper {
        return ActivityHelper()
    }
}
```

### Testing with Hilt

Create a test module:

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [DatabaseModule::class]
)
object TestDatabaseModule {
    
    @Provides
    @Singleton
    fun provideInMemoryDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.inMemoryDatabaseBuilder(
            context,
            AppDatabase::class.java
        )
            .allowMainThreadQueries() // Only for testing
            .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}

@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class UserRepositoryTest {
    
    @get:Rule
    var hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var database: AppDatabase
    
    @Inject
    lateinit var repository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @After
    fun tearDown() {
        database.close()
    }
    
    @Test
    fun insertUser_retrievesUser() = runBlocking {
        val user = User(firstName = "John", lastName = "Doe", email = "john@test.com", age = 30)
        val id = repository.insert(user)
        
        val retrieved = repository.getUserById(id).getOrAwaitValue()
        assertEquals(user.firstName, retrieved?.firstName)
    }
}
```

### Complete Architecture Example

```kotlin
// Domain Layer - Use Cases
class GetAllUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    operator fun invoke(): LiveData<List<User>> = repository.allUsers
}

class UpdateUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(user: User) {
        repository.update(user)
    }
}

// Data Layer - Repository
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val remoteDataSource: UserRemoteDataSource,
    @IoDispatcher private val dispatcher: CoroutineDispatcher
) {
    val allUsers: LiveData<List<User>> = userDao.getAllUsers()
    
    suspend fun refreshUsers() = withContext(dispatcher) {
        try {
            val remoteUsers = remoteDataSource.fetchUsers()
            userDao.insertAll(remoteUsers)
        } catch (e: Exception) {
            // Handle error
        }
    }
    
    suspend fun update(user: User) = withContext(dispatcher) {
        userDao.updateUser(user)
    }
}

// Presentation Layer - ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getAllUsersUseCase: GetAllUsersUseCase,
    private val updateUserUseCase: UpdateUserUseCase
) : ViewModel() {
    
    val users: LiveData<List<User>> = getAllUsersUseCase()
    
    fun updateUser(user: User) = viewModelScope.launch {
        updateUserUseCase(user)
    }
}

// UI Layer
@AndroidEntryPoint
class UserListFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewModel.users.observe(viewLifecycleOwner) { users ->
            // Update UI
        }
    }
}
```

### Dispatchers Module

Provide coroutine dispatchers:

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatchersModule {
    
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
    
    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

// Usage
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun loadUsers() = withContext(ioDispatcher) {
        userDao.getAllUsersOnce()
    }
}
```

## Transactions

For multiple operations that should succeed or fail together:

```kotlin
@Dao
interface UserDao {
    @Transaction
    suspend fun updateUserAndInsertLog(user: User, log: ActivityLog) {
        updateUser(user)
        insertLog(log)
    }
    
    @Update
    suspend fun updateUser(user: User)
    
    @Insert
    suspend fun insertLog(log: ActivityLog)
}
```

## Testing

### Unit Testing with In-Memory Database

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao
    
    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(
            context,
            AppDatabase::class.java
        ).build()
        userDao = database.userDao()
    }
    
    @After
    fun tearDown() {
        database.close()
    }
    
    @Test
    fun insertAndRetrieveUser() = runBlocking {
        val user = User(firstName = "John", lastName = "Doe", email = "john@example.com", age = 30)
        val id = userDao.insertUser(user)
        
        val retrieved = userDao.getUserById(id)
        assertEquals(user.firstName, retrieved?.firstName)
    }
}

// Helper function for testing LiveData
fun <T> LiveData<T>.getOrAwaitValue(
    time: Long = 2,
    timeUnit: TimeUnit = TimeUnit.SECONDS
): T {
    var data: T? = null
    val latch = CountDownLatch(1)
    val observer = object : Observer<T> {
        override fun onChanged(value: T) {
            data = value
            latch.countDown()
            this@getOrAwaitValue.removeObserver(this)
        }
    }
    this.observeForever(observer)
    
    if (!latch.await(time, timeUnit)) {
        throw TimeoutException("LiveData value was never set.")
    }
    
    @Suppress("UNCHECKED_CAST")
    return data as T
}

// Usage in tests
@Test
fun getAllUsers_returnsAllUsers() = runBlocking {
    val user1 = User(firstName = "John", lastName = "Doe", email = "john@test.com", age = 30)
    val user2 = User(firstName = "Jane", lastName = "Smith", email = "jane@test.com", age = 25)
    
    userDao.insertUser(user1)
    userDao.insertUser(user2)
    
    val users = userDao.getAllUsers().getOrAwaitValue()
    assertEquals(2, users.size)
}
```


## Best Practices

1. **Use Coroutines**: Make database operations suspending functions to avoid blocking the main thread
2. **Repository Pattern**: Create a repository layer between your ViewModel and DAO
3. **Singleton Database**: Use a singleton pattern for the database instance
4. **Handle Migrations**: Always provide migration paths for production apps
5. **Use Flow or LiveData**: Observe data changes reactively
6. **Indexing**: Add indices to columns you frequently query
7. **Avoid Main Thread**: Never perform database operations on the main thread in production
8. **Export Schema**: Set `exportSchema = true` and specify a directory for schema tracking
9. **Use @Transaction**: For operations that should be atomic
10. **Type Converters**: Keep them simple and efficient

## Advanced Features

### Indexing

```kotlin
@Entity(
    tableName = "users",
    indices = [Index(value = ["email"], unique = true)]
)
data class User(
    @PrimaryKey val id: Long,
    val email: String,
    val name: String
)
```

### Prepopulating Database

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .createFromAsset("database/myapp.db")
    .build()
```

### Raw Queries

```kotlin
@Dao
interface UserDao {
    @RawQuery
    fun getUsersViaQuery(query: SupportSQLiteQuery): List<User>
}

// Usage
val query = SimpleSQLiteQuery("SELECT * FROM users WHERE age > ? ORDER BY age ASC", arrayOf(18))
val users = userDao.getUsersViaQuery(query)
```

## Common Pitfalls

1. **Main Thread Operations**: Room doesn't allow database operations on the main thread by default
2. **Missing Migrations**: Not providing migrations causes data loss
3. **Complex Objects**: Forgetting to use Type Converters for non-primitive types
4. **Memory Leaks**: Not properly scoping database instances
5. **Query Errors**: Runtime crashes from SQL syntax errors (use compile-time verification)

## Resources

- [Official Room Documentation](https://developer.android.com/training/data-storage/room)
- [Room Database Migration Guide](https://developer.android.com/training/data-storage/room/migrating-db-versions)
- [Codelab: Android Room with a View](https://developer.android.com/codelabs/android-room-with-a-view)

---

This guide covers the essentials of Room. For more advanced use cases, refer to the official Android documentation.