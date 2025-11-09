# Complete Android Retrofit Guide

**Author:** MiniMax Agent  
**Date:** November 9, 2025

## Table of Contents

1. [Introduction to Retrofit](#introduction-to-retrofit)
2. [Installation and Setup](#installation-and-setup)
3. [Basic Concepts](#basic-concepts)
4. [API Declaration](#api-declaration)
5. [HTTP Methods](#http-methods)
6. [Headers](#headers)
7. [Query Parameters](#query-parameters)
8. [Request Body](#request-body)
9. [Response Handling](#response-handling)
10. [Error Handling](#error-handling)
11. [Advanced Features](#advanced-features)
12. [Coroutine Integration](#coroutine-integration)
13. [Practical Examples](#practical-examples)
14. [Best Practices](#best-practices)

---

## Introduction to Retrofit

Retrofit is a type-safe HTTP client for Android and Java applications. It simplifies the process of making HTTP requests to RESTful APIs by providing a clean, declarative API that integrates seamlessly with your codebase.

### Key Features

- **Type Safety**: Automatic parsing of JSON responses into Java/Kotlin objects
- **Converter Support**: Built-in support for various serialization libraries
- **Coroutine Support**: Native support for Kotlin Coroutines
- **Interceptors**: Request/response logging, authentication, and more
- **Error Handling**: Comprehensive error handling mechanisms
- **Testability**: Easy to mock and test

---

## Installation and Setup

### Adding Dependencies

First, add the required dependencies to your `app/build.gradle` file:

```kotlin
dependencies {
    // Retrofit
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    
    // Gson converter (most common)
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    
    // Or other converters
    implementation 'com.squareup.retrofit2:converter-jackson:2.9.0'
    implementation 'com.squareup.retrofit2:converter-moshi:2.9.0'
    
    // OkHttp (Retrofit's underlying HTTP client)
    implementation 'com.squareup.okhttp3:okhttp:4.11.0'
    
    // Coroutines support
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    
    // Logging interceptor
    implementation 'com.squareup.okhttp3:logging-interceptor:4.11.0'
}
```

### Internet Permission

Add the following permission to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

---

## Basic Concepts

### Understanding the Architecture

Retrofit follows a layered architecture:

1. **Interface**: Defines the API contract
2. **Retrofit Instance**: Configuration and builder
3. **Service Interface**: Concrete implementation of the interface
4. **Call/Response**: HTTP request/response objects

### Simple Retrofit Setup

```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

class ApiClient {
    companion object {
        // Base URL for your API - MUST end with /
        private const val BASE_URL = "https://api.example.com/"
        
        // Create a singleton Retrofit instance
        val instance: ApiService by lazy {
            val retrofit = Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create()) // JSON to Kotlin objects
                .build()
            
            retrofit.create(ApiService::class.java)
        }
    }
}
```

---

## API Declaration

### Defining API Interfaces

The core of Retrofit is defining interfaces with method annotations that describe HTTP requests:

```kotlin
import retrofit2.Call
import retrofit2.http.*

// Define your data models first
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val phone: String
)

data class Post(
    val userId: Int,
    val id: Int,
    val title: String,
    val body: String
)

// Create API service interface
interface ApiService {
    
    // GET request to fetch a user by ID
    @GET("users/{id}")
    fun getUser(@Path("id") userId: Int): Call<User>
    
    // GET request to fetch all posts
    @GET("posts")
    fun getPosts(): Call<List<Post>>
    
    // POST request to create a new user
    @POST("users")
    fun createUser(@Body user: User): Call<User>
    
    // PUT request to update a user
    @PUT("users/{id}")
    fun updateUser(@Path("id") userId: Int, @Body user: User): Call<User>
    
    // DELETE request to remove a user
    @DELETE("users/{id}")
    fun deleteUser(@Path("id") userId: Int): Call<Void>
}
```

---

## HTTP Methods

### GET Requests

#### Basic GET

```kotlin
@GET("users")
fun getAllUsers(): Call<List<User>>

@GET("users/{id}")
fun getUserById(@Path("id") id: Int): Call<User>
```

#### GET with Multiple Parameters

```kotlin
@GET("users")
fun getUsers(
    @Query("page") page: Int,
    @Query("per_page") perPage: Int,
    @Query("sort") sort: String
): Call<List<User>>
```

#### GET with Map Parameters

```kotlin
@GET("search/users")
fun searchUsers(@QueryMap params: Map<String, String>): Call<List<User>>

// Usage
val searchParams = mapOf(
    "q" to "john",
    "sort" to "name",
    "order" to "asc"
)
```

### POST Requests

#### POST with JSON Body

```kotlin
@POST("users")
fun createUser(@Body user: User): Call<User>

// Usage
val newUser = User(0, "John Doe", "john@example.com", "1234567890")
val call = apiService.createUser(newUser)
```

#### POST with Form Data

```kotlin
@FormUrlEncoded
@POST("users")
fun createUserForm(
    @Field("name") name: String,
    @Field("email") email: String,
    @Field("phone") phone: String
): Call<User>
```

### PUT and PATCH Requests

```kotlin
// Complete update
@PUT("users/{id}")
fun updateUser(@Path("id") id: Int, @Body user: User): Call<User>

// Partial update
@PATCH("users/{id}")
fun partialUpdateUser(
    @Path("id") id: Int,
    @Field("email") email: String
): Call<User>
```

### DELETE Requests

```kotlin
@DELETE("users/{id}")
fun deleteUser(@Path("id") id: Int): Call<Void>
```

---

## Headers

### Static Headers

```kotlin
@Headers("Accept: application/json")
@GET("users")
fun getUsers(): Call<List<User>>

// Multiple headers
@Headers(
    "Accept: application/json",
    "User-Agent: MyApp/1.0"
)
@GET("users")
fun getUsers(): Call<List<User>>
```

### Dynamic Headers

```kotlin
@GET("users")
fun getUsers(
    @Header("Authorization") authToken: String
): Call<List<User>>

// Optional header
@GET("users")
fun getUsersOptional(
    @HeaderMap headers: Map<String, String>
): Call<List<User>>
```

### Reusable Header Interceptor

```kotlin
class AuthInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val newRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer your_token_here")
            .build()
        return chain.proceed(newRequest)
    }
}

// Add to Retrofit builder
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .client(
        OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .build()
    )
    .build()
```

---

## Query Parameters

### Required Parameters

```kotlin
@GET("users")
fun getUsersByCountry(@Query("country") country: String): Call<List<User>>

@GET("users")
fun getUsersWithPagination(
    @Query("page") page: Int,
    @Query("limit") limit: Int
): Call<List<User>>
```

### Optional Parameters

```kotlin
@GET("users")
fun getUsers(
    @Query("country") country: String? = null,
    @Query("age") age: Int? = null,
    @Query("status") status: String? = null
): Call<List<User>>

// Usage
val users = apiService.getUsers(country = "USA", age = 25)
```

### Query Maps

```kotlin
@GET("search/users")
fun searchUsers(@QueryMap filters: Map<String, String>): Call<List<User>>

// Usage
val filters = mutableMapOf<String, String>().apply {
    put("country", "USA")
    put("age_min", "18")
    put("age_max", "65")
    put("status", "active")
}
val call = apiService.searchUsers(filters)
```

### Complex Query Parameters

```kotlin
// Array parameters
@GET("users")
fun getUsersByIds(@Query("ids") ids: List<Int>): Call<List<User>>

// Usage
val userIds = listOf(1, 2, 3, 4, 5)
val call = apiService.getUsersByIds(userIds)

// Boolean parameters
@GET("users")
fun getActiveUsers(@Query("active") isActive: Boolean): Call<List<User>>

// Usage
val activeUsers = apiService.getActiveUsers(true)
```

---

## Request Body

### JSON Request Body

```kotlin
// Using data classes
data class CreateUserRequest(
    val name: String,
    val email: String,
    val phone: String
)

@POST("users")
fun createUser(@Body request: CreateUserRequest): Call<User>

// Usage
val request = CreateUserRequest(
    name = "John Doe",
    email = "john@example.com",
    phone = "1234567890"
)
val call = apiService.createUser(request)
```

### Raw JSON Body

```kotlin
@POST("users")
fun createUserRaw(@Body json: String): Call<User>

// Usage
val jsonBody = """
    {
        "name": "John Doe",
        "email": "john@example.com",
        "phone": "1234567890"
    }
"""
val call = apiService.createUserRaw(jsonBody)
```

### Multipart Form Data

```kotlin
@Multipart
@POST("users")
fun createUserWithPhoto(
    @Part("name") name: RequestBody,
    @Part("email") email: RequestBody,
    @Part("phone") phone: RequestBody,
    @Part profileImage: MultipartBody.Part
): Call<User>

// Usage
val name = RequestBody.create("text/plain".toMediaTypeOrNull(), "John Doe")
val email = RequestBody.create("text/plain".toMediaTypeOrNull(), "john@example.com")
val phone = RequestBody.create("text/plain".toMediaTypeOrNull(), "1234567890")

val imageFile = File("path/to/profile.jpg")
val imageRequestBody = imageFile.asRequestBody("image/jpeg".toMediaTypeOrNull())
val imagePart = MultipartBody.Part.createFormData(
    "profileImage",
    imageFile.name,
    imageRequestBody
)

val call = apiService.createUserWithPhoto(name, email, phone, imagePart)
```

---

## Response Handling

### Basic Response Handling

```kotlin
import retrofit2.Callback
import retrofit2.Response

class UserRepository {
    private val apiService = ApiClient.instance
    
    fun getUser(userId: Int, callback: UserCallback) {
        val call = apiService.getUser(userId)
        
        // Execute the request asynchronously
        call.enqueue(object : Callback<User> {
            override fun onResponse(call: Call<User>, response: Response<User>) {
                if (response.isSuccessful) {
                    val user = response.body()
                    if (user != null) {
                        callback.onSuccess(user)
                    } else {
                        callback.onError("Empty response body")
                    }
                } else {
                    // Handle HTTP error (4xx, 5xx)
                    callback.onError("Error ${response.code()}: ${response.message()}")
                }
            }
            
            override fun onFailure(call: Call<User>, t: Throwable) {
                // Handle network failure
                callback.onError("Network error: ${t.message}")
            }
        })
    }
}

// Define callback interface
interface UserCallback {
    fun onSuccess(user: User)
    fun onError(message: String)
}
```

### Response Wrapper Classes

```kotlin
// Common wrapper for API responses
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val message: String? = null,
    val errorCode: Int? = null
)

// In your API service
@GET("users/{id}")
fun getUser(@Path("id") id: Int): Call<ApiResponse<User>>

// Or for paginated responses
data class PaginatedResponse<T>(
    val data: List<T>,
    val total: Int,
    val page: Int,
    val perPage: Int,
    val totalPages: Int
)

@GET("users")
fun getUsers(@Query("page") page: Int): Call<PaginatedResponse<User>>
```

### Custom Response Types

```kotlin
// Success response with data
data class SuccessResponse<T>(val data: T)

// Error response
data class ErrorResponse(
    val message: String,
    val code: Int,
    val details: String? = null
)

// Generic response handling
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val exception: Throwable? = null) : ApiResult<Nothing>()
    data class Loading(val isLoading: Boolean) : ApiResult<Nothing>()
}

// Extension function for easier handling
fun <T> Call<T>.execute(): ApiResult<T> {
    return try {
        val response = this.execute()
        if (response.isSuccessful) {
            response.body()?.let { ApiResult.Success(it) }
                ?: ApiResult.Error("Empty response body")
        } else {
            ApiResult.Error("HTTP ${response.code()}: ${response.message()}")
        }
    } catch (e: Exception) {
        ApiResult.Error("Network error: ${e.message}", e)
    }
}
```

---

## Error Handling

### HTTP Error Status Codes

```kotlin
import retrofit2.HttpException
import java.io.IOException
import java.net.SocketTimeoutException

class ApiErrorHandler {
    
    fun handleError(throwable: Throwable): ApiError {
        return when (throwable) {
            is HttpException -> {
                val code = throwable.code()
                val message = when (code) {
                    400 -> "Bad Request - Please check your input"
                    401 -> "Unauthorized - Please log in"
                    403 -> "Forbidden - You don't have permission"
                    404 -> "Not Found - Resource doesn't exist"
                    408 -> "Request Timeout - Please try again"
                    409 -> "Conflict - Resource already exists"
                    422 -> "Unprocessable Entity - Invalid data format"
                    500 -> "Internal Server Error - Please try later"
                    502 -> "Bad Gateway - Service temporarily unavailable"
                    503 -> "Service Unavailable - Please try later"
                    else -> "HTTP $code - ${throwable.message()}"
                }
                ApiError(code, message, throwable)
            }
            is IOException -> {
                ApiError(-1, "Network error - Please check your connection", throwable)
            }
            is SocketTimeoutException -> {
                ApiError(-2, "Request timeout - Please try again", throwable)
            }
            else -> {
                ApiError(-3, "Unknown error: ${throwable.message}", throwable)
            }
        }
    }
}

data class ApiError(
    val code: Int,
    val message: String,
    val exception: Throwable
)
```

### Comprehensive Error Handling

```kotlin
class UserRepository {
    private val apiService = ApiClient.instance
    private val errorHandler = ApiErrorHandler()
    
    // Using coroutines for better async handling
    suspend fun getUser(userId: Int): Result<User> {
        return try {
            val response = apiService.getUser(userId).await()
            if (response.isSuccessful) {
                response.body()?.let { Result.success(it) }
                    ?: Result.failure(Exception("Empty response body"))
            } else {
                val error = errorHandler.handleError(
                    HttpException(response)
                )
                Result.failure(Exception(error.message))
            }
        } catch (e: Exception) {
            val error = errorHandler.handleError(e)
            Result.failure(Exception(error.message))
        }
    }
    
    // Repository with LiveData for UI updates
    fun getUserLiveData(userId: Int): LiveData<Resource<User>> {
        val liveData = MutableLiveData<Resource<User>>()
        liveData.value = Resource.Loading()
        
        apiService.getUser(userId).enqueue(object : Callback<User> {
            override fun onResponse(call: Call<User>, response: Response<User>) {
                if (response.isSuccessful) {
                    liveData.value = Resource.Success(response.body()!!)
                } else {
                    val error = errorHandler.handleError(HttpException(response))
                    liveData.value = Resource.Error(error.message)
                }
            }
            
            override fun onFailure(call: Call<User>, t: Throwable) {
                val error = errorHandler.handleError(t)
                liveData.value = Resource.Error(error.message)
            }
        })
        
        return liveData
    }
}

// Resource class for state management
sealed class Resource<out T> {
    object Loading : Resource<Nothing>()
    data class Success<out T>(val data: T) : Resource<T>()
    data class Error(val message: String) : Resource<Nothing>()
}
```

---

## Advanced Features

### Authentication Interceptors

```kotlin
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        tokenProvider()?.let { token ->
            val newRequest = originalRequest.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
            return chain.proceed(newRequest)
        }
        
        return chain.proceed(originalRequest)
    }
}

// Token manager
class TokenManager(private val context: Context) {
    private val sharedPrefs = context.getSharedPreferences("auth", Context.MODE_PRIVATE)
    
    fun saveToken(token: String) {
        sharedPrefs.edit().putString("access_token", token).apply()
    }
    
    fun getToken(): String? = sharedPrefs.getString("access_token", null)
    
    fun clearToken() {
        sharedPrefs.edit().remove("access_token").apply()
    }
}

// Setup Retrofit with authentication
val tokenManager = TokenManager(context)
val authInterceptor = AuthInterceptor { tokenManager.getToken() }

val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(authInterceptor)
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

### Caching Strategy

```kotlin
class CacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)
        
        // Cache successful GET requests for 5 minutes
        if (request.method == "GET" && response.isSuccessful) {
            val cacheControl = CacheControl.Builder()
                .maxAge(5, TimeUnit.MINUTES)
                .build()
            return response.newBuilder()
                .header("Cache-Control", cacheControl.toString())
                .build()
        }
        
        return response
    }
}

// Configure caching
val cacheSize = 10 * 1024 * 1024 // 10 MB
val cache = Cache(context.cacheDir, cacheSize.toLong())

val okHttpClient = OkHttpClient.Builder()
    .cache(cache)
    .addInterceptor(CacheInterceptor())
    .build()
```

### Rate Limiting

```kotlin
class RateLimitInterceptor : Interceptor {
    private val callCount = AtomicInteger(0)
    private val rateLimit = 5 // 5 calls per minute
    private val timeWindow = 60_000L // 1 minute in milliseconds
    private val lastResetTime = AtomicLong(System.currentTimeMillis())
    
    override fun intercept(chain: Interceptor.Chain): Response {
        if (!isWithinRateLimit()) {
            throw RateLimitException("Rate limit exceeded. Please wait before making more requests.")
        }
        
        return chain.proceed(chain.request())
    }
    
    private fun isWithinRateLimit(): Boolean {
        val currentTime = System.currentTimeMillis()
        val lastReset = lastResetTime.get()
        
        // Reset counter if time window has passed
        if (currentTime - lastReset >= timeWindow) {
            callCount.set(0)
            lastResetTime.set(currentTime)
        }
        
        return callCount.getAndIncrement() < rateLimit
    }
}

class RateLimitException(message: String) : Exception(message)
```

---

## Coroutine Integration

### Suspending Functions

```kotlin
interface ApiService {
    // Using suspend functions instead of Call
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): Response<User>
    
    @GET("users")
    suspend fun getUsers(): Response<List<User>>
    
    @POST("users")
    suspend fun createUser(@Body user: User): Response<User>
    
    // With query parameters
    @GET("users")
    suspend fun getUsers(
        @Query("page") page: Int,
        @Query("limit") limit: Int
    ): Response<List<User>>
}
```

### Repository Pattern with Coroutines

```kotlin
class UserRepository {
    private val apiService = ApiClient.instance
    
    suspend fun getUser(userId: Int): User {
        val response = apiService.getUser(userId)
        if (response.isSuccessful) {
            return response.body() ?: throw Exception("Empty response body")
        } else {
            throw HttpException(response)
        }
    }
    
    suspend fun getUsers(page: Int = 1, limit: Int = 20): List<User> {
        val response = apiService.getUsers(page, limit)
        if (response.isSuccessful) {
            return response.body() ?: emptyList()
        } else {
            throw HttpException(response)
        }
    }
    
    suspend fun createUser(user: User): User {
        val response = apiService.createUser(user)
        if (response.isSuccessful) {
            return response.body() ?: throw Exception("Empty response body")
        } else {
            throw HttpException(response)
        }
    }
}
```

### ViewModel with Coroutines

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class UserViewModel : ViewModel() {
    private val userRepository = UserRepository()
    
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    fun loadUser(userId: Int) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            try {
                val user = userRepository.getUser(userId)
                _uiState.value = UserUiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UserUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun createUser(name: String, email: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            try {
                val user = userRepository.createUser(User(0, name, email, ""))
                _uiState.value = UserUiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UserUiState.Error(e.message ?: "Failed to create user")
            }
        }
    }
}

sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String) : UserUiState()
}
```

### Fragment/Activity with Coroutines

```kotlin
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

class UserFragment : Fragment() {
    private lateinit var viewModel: UserViewModel
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewModel = ViewModelProvider(this)[UserViewModel::class.java]
        
        // Collect UI state
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is UserUiState.Loading -> showLoading()
                    is UserUiState.Success -> showUser(state.user)
                    is UserUiState.Error -> showError(state.message)
                }
            }
        }
        
        // Load user when fragment is ready
        viewModel.loadUser(1)
    }
    
    private fun showLoading() {
        // Show progress indicator
        binding.progressBar.visibility = View.VISIBLE
        binding.userInfo.visibility = View.GONE
        binding.errorText.visibility = View.GONE
    }
    
    private fun showUser(user: User) {
        // Display user information
        binding.progressBar.visibility = View.GONE
        binding.userInfo.visibility = View.VISIBLE
        binding.errorText.visibility = View.GONE
        
        binding.nameText.text = user.name
        binding.emailText.text = user.email
    }
    
    private fun showError(message: String) {
        // Display error message
        binding.progressBar.visibility = View.GONE
        binding.userInfo.visibility = View.GONE
        binding.errorText.visibility = View.VISIBLE
        binding.errorText.text = message
    }
}
```

---

## Practical Examples

### Complete User Management App

```kotlin
// Data Models
data class User(
    val id: Int = 0,
    val name: String = "",
    val email: String = "",
    val phone: String = "",
    val createdAt: String? = null,
    val updatedAt: String? = null
)

data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val message: String? = null,
    val errors: List<String>? = null
)

data class LoginRequest(
    val email: String,
    val password: String
)

data class LoginResponse(
    val accessToken: String,
    val refreshToken: String,
    val user: User
)

// API Service
interface UserApiService {
    @POST("auth/login")
    suspend fun login(@Body request: LoginRequest): Response<ApiResponse<LoginResponse>>
    
    @POST("auth/refresh")
    suspend fun refreshToken(@Body refreshToken: String): Response<ApiResponse<LoginResponse>>
    
    @GET("users")
    suspend fun getUsers(@Query("page") page: Int = 1): Response<ApiResponse<List<User>>>
    
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): Response<ApiResponse<User>>
    
    @POST("users")
    suspend fun createUser(@Body user: User): Response<ApiResponse<User>>
    
    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Int, @Body user: User): Response<ApiResponse<User>>
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Int): Response<ApiResponse<User>>
}

// Repository with comprehensive error handling
class UserRepository {
    private val apiService = ApiClient.instance.userApiService
    private val tokenManager = TokenManager.getInstance()
    
    suspend fun login(email: String, password: String): Result<User> {
        return try {
            val request = LoginRequest(email, password)
            val response = apiService.login(request)
            
            if (response.isSuccessful) {
                val apiResponse = response.body()
                if (apiResponse?.success == true) {
                    apiResponse.data?.let { loginResponse ->
                        tokenManager.saveTokens(
                            loginResponse.accessToken,
                            loginResponse.refreshToken
                        )
                        Result.success(loginResponse.user)
                    } ?: Result.failure(Exception("Login response is empty"))
                } else {
                    Result.failure(Exception(apiResponse?.message ?: "Login failed"))
                }
            } else {
                Result.failure(Exception("HTTP ${response.code()}: ${response.message()}"))
            }
        } catch (e: Exception) {
            Result.failure(Exception("Network error: ${e.message}"))
        }
    }
    
    suspend fun getUsers(page: Int = 1): Result<List<User>> {
        return safeApiCall {
            val response = apiService.getUsers(page)
            if (response.isSuccessful) {
                response.body()?.data ?: emptyList()
            } else {
                throw Exception("Failed to fetch users")
            }
        }
    }
    
    suspend fun createUser(user: User): Result<User> {
        return safeApiCall {
            val response = apiService.createUser(user)
            if (response.isSuccessful) {
                response.body()?.data ?: throw Exception("User creation failed")
            } else {
                throw Exception("Failed to create user")
            }
        }
    }
    
    private suspend fun <T> safeApiCall(apiCall: suspend () -> T): Result<T> {
        return try {
            val result = apiCall()
            Result.success(result)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// Token Manager
object TokenManager {
    private const val PREFS_NAME = "auth_prefs"
    private const val ACCESS_TOKEN_KEY = "access_token"
    private const val REFRESH_TOKEN_KEY = "refresh_token"
    
    private lateinit var context: Context
    private val prefs: SharedPreferences by lazy {
        context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
    }
    
    fun initialize(context: Context) {
        this.context = context.applicationContext
    }
    
    fun saveTokens(accessToken: String, refreshToken: String) {
        prefs.edit()
            .putString(ACCESS_TOKEN_KEY, accessToken)
            .putString(REFRESH_TOKEN_KEY, refreshToken)
            .apply()
    }
    
    fun getAccessToken(): String? = prefs.getString(ACCESS_TOKEN_KEY, null)
    
    fun getRefreshToken(): String? = prefs.getString(REFRESH_TOKEN_KEY, null)
    
    fun clearTokens() {
        prefs.edit()
            .remove(ACCESS_TOKEN_KEY)
            .remove(REFRESH_TOKEN_KEY)
            .apply()
    }
    
    fun isLoggedIn(): Boolean = !getAccessToken().isNullOrEmpty()
}

// Retrofit Client
object ApiClient {
    private const val BASE_URL = "https://api.example.com/"
    private const val TIMEOUT_SECONDS = 30L
    
    val instance: UserApiService by lazy {
        val logging = HttpLoggingInterceptor()
        logging.setLevel(HttpLoggingInterceptor.Level.BODY)
        
        val authInterceptor = AuthInterceptor()
        
        val okHttpClient = OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addInterceptor(logging)
            .connectTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .readTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .writeTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .build()
        
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(UserApiService::class.java)
    }
}
```

### File Upload Example

```kotlin
interface FileUploadService {
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part("description") description: RequestBody,
        @Part file: MultipartBody.Part
    ): Response<ApiResponse<UploadResponse>>
}

class FileUploadRepository {
    private val apiService = ApiClient.fileUploadService
    
    suspend fun uploadFile(context: Context, fileUri: Uri, description: String): Result<UploadResult> {
        return try {
            // Convert URI to File
            val file = File(getRealPathFromUri(context, fileUri) ?: throw Exception("File not found"))
            
            // Create request body
            val requestFile = file.asRequestBody("image/*".toMediaTypeOrNull())
            val filePart = MultipartBody.Part.createFormData(
                "file",
                file.name,
                requestFile
            )
            
            // Create description request body
            val descriptionPart = RequestBody.create("text/plain".toMediaTypeOrNull(), description)
            
            // Make API call
            val response = apiService.uploadFile(descriptionPart, filePart)
            
            if (response.isSuccessful) {
                val apiResponse = response.body()
                if (apiResponse?.success == true) {
                    apiResponse.data?.let { uploadResponse ->
                        Result.success(
                            UploadResult.Success(
                                fileName = uploadResponse.fileName,
                                fileUrl = uploadResponse.url,
                                message = uploadResponse.message
                            )
                        )
                    } ?: Result.failure(Exception("Upload response is empty"))
                } else {
                    Result.failure(Exception(apiResponse?.message ?: "Upload failed"))
                }
            } else {
                Result.failure(Exception("HTTP ${response.code()}: ${response.message()}"))
            }
        } catch (e: Exception) {
            Result.failure(Exception("Upload error: ${e.message}"))
        }
    }
    
    private fun getRealPathFromUri(context: Context, uri: Uri): String? {
        var cursor: Cursor? = null
        return try {
            val projection = arrayOf(MediaStore.Images.Media.DATA)
            cursor = context.contentResolver.query(uri, projection, null, null, null)
            val columnIndex = cursor?.getColumnIndexOrThrow(MediaStore.Images.Media.DATA)
            cursor?.moveToFirst()
            columnIndex?.let { index -> cursor?.getString(index) }
        } finally {
            cursor?.close()
        }
    }
}

data class UploadResponse(
    val fileName: String,
    val url: String,
    val message: String
)

sealed class UploadResult {
    data class Success(
        val fileName: String,
        val fileUrl: String,
        val message: String
    ) : UploadResult()
    
    data class Error(val message: String) : UploadResult()
}
```

### Pagination Implementation

```kotlin
interface PaginatedApiService {
    @GET("posts")
    suspend fun getPosts(
        @Query("page") page: Int,
        @Query("limit") limit: Int
    ): Response<ApiResponse<PaginatedResponse<Post>>>
}

class PaginatedRepository {
    private val apiService = ApiClient.paginatedApiService
    
    private val cache = LinkedHashMap<Int, List<Post>>()
    private var currentPage = 1
    private var isLoading = false
    private var hasMore = true
    
    suspend fun getPosts(page: Int = 1, limit: Int = 20): PagedResult<List<Post>> {
        if (isLoading) return PagedResult.Loading
        
        isLoading = true
        
        return try {
            val response = apiService.getPosts(page, limit)
            
            if (response.isSuccessful) {
                val apiResponse = response.body()
                if (apiResponse?.success == true) {
                    val paginatedResponse = apiResponse.data
                    if (paginatedResponse != null) {
                        cache[page] = paginatedResponse.data
                        currentPage = page
                        hasMore = page < paginatedResponse.totalPages
                        
                        isLoading = false
                        
                        PagedResult.Success(
                            data = paginatedResponse.data,
                            hasMore = hasMore,
                            currentPage = currentPage
                        )
                    } else {
                        isLoading = false
                        PagedResult.Error("No data available")
                    }
                } else {
                    isLoading = false
                    PagedResult.Error(apiResponse?.message ?: "Failed to fetch posts")
                }
            } else {
                isLoading = false
                PagedResult.Error("HTTP ${response.code()}: ${response.message()}")
            }
        } catch (e: Exception) {
            isLoading = false
            PagedResult.Error("Network error: ${e.message}")
        }
    }
    
    fun getCachedPosts(page: Int): List<Post>? = cache[page]
    
    fun isHasMore(): Boolean = hasMore
    fun isLoading(): Boolean = isLoading
    fun getCurrentPage(): Int = currentPage
}

sealed class PagedResult<out T> {
    data class Success<T>(
        val data: T,
        val hasMore: Boolean,
        val currentPage: Int
    ) : PagedResult<T>()
    
    data class Error(val message: String) : PagedResult<T>()
    object Loading : PagedResult<Nothing>()
}
```

---

## Best Practices

### 1. Repository Pattern

Always use the Repository pattern to separate API calls from UI logic:

```kotlin
class UserRepository {
    private val apiService = ApiClient.instance
    private val cacheManager = CacheManager()
    
    // Single source of truth
    suspend fun getUser(userId: Int): User {
        // Try cache first
        cacheManager.getUser(userId)?.let { return it }
        
        // If not in cache, fetch from API
        val user = apiService.getUser(userId).await()
        cacheManager.saveUser(user)
        return user
    }
}
```

### 2. Error Handling

Implement comprehensive error handling at the repository level:

```kotlin
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: Int? = null) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}
```

### 3. Dependency Injection

Use dependency injection to manage Retrofit instances:

```kotlin
// Using Hilt or Dagger
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            })
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}
```

### 4. Caching Strategy

Implement appropriate caching for better performance:

```kotlin
class CacheManager(private val context: Context) {
    private val prefs = context.getSharedPreferences("cache", Context.MODE_PRIVATE)
    
    fun <T> saveData(key: String, data: T) {
        val json = Gson().toJson(data)
        prefs.edit().putString(key, json).apply()
    }
    
    fun <T> getData(key: String, type: Class<T>): T? {
        val json = prefs.getString(key, null) ?: return null
        return try {
            Gson().fromJson(json, type)
        } catch (e: Exception) {
            null
        }
    }
}
```

### 5. Testing

Write unit tests for your API services and repositories:

```kotlin
@RunWith(MockitoJUnitRunner::class)
class UserRepositoryTest {
    
    @Mock
    private lateinit var mockApiService: UserApiService
    
    private lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        userRepository = UserRepository(mockApiService)
    }
    
    @Test
    fun `getUser should return user when API call is successful`() = runTest {
        // Given
        val userId = 1
        val expectedUser = User(id = userId, name = "John Doe", email = "john@example.com")
        `when`(mockApiService.getUser(userId)).thenReturn(Response.success(expectedUser))
        
        // When
        val result = userRepository.getUser(userId)
        
        // Then
        assertEquals(expectedUser, result)
        verify(mockApiService).getUser(userId)
    }
}
```

### 6. Security Considerations

- Never hardcode API keys or sensitive information
- Use proper authentication tokens
- Implement certificate pinning for sensitive applications
- Validate and sanitize all user inputs

```kotlin
class SecurityInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        // Add security headers
        val secureRequest = originalRequest.newBuilder()
            .addHeader("X-Requested-With", "XMLHttpRequest")
            .addHeader("X-Content-Type-Options", "nosniff")
            .build()
        
        return chain.proceed(secureRequest)
    }
}
```

### 7. Performance Optimization

- Use appropriate timeouts
- Implement connection pooling
- Cache responses when appropriate
- Use pagination for large datasets

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(5, 5, TimeUnit.MINUTES))
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .build()
```

---

This comprehensive guide covers all aspects of Retrofit usage in Android development. Each example includes detailed comments and explanations to help you understand the concepts and implement them effectively in your own projects.

Remember to always handle errors gracefully, implement proper authentication, and follow Android best practices when integrating Retrofit into your applications.
