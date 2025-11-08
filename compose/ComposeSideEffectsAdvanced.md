# Complete Guide to Jetpack Compose Side Effects

A comprehensive tutorial covering all aspects of side effects in Jetpack Compose with practical examples and real-world use cases.

## Table of Contents
- [Lesson 1: Introduction to Side Effects](#lesson-1-introduction-to-side-effects)
- [Lesson 2: LaunchedEffect - One-time Side Effects](#lesson-2-launchedeffect---one-time-side-effects)
- [Lesson 3: DisposableEffect - Cleanup Side Effects](#lesson-3-disposableeffect---cleanup-side-effects)
- [Lesson 4: RememberCoroutineScope - Coroutine Scope Management](#lesson-4-remembercoroutinescope---coroutine-scope-management)
- [Lesson 5: RememberUpdatedState - Preserving Latest Values](#lesson-5-rememberupdatedstate---preserving-latest-values)
- [Lesson 6: SideEffect - Every Recomposition](#lesson-6-sideeffect---every-recomposition)
- [Lesson 7: LaunchedEffect with Keys - Dynamic Side Effects](#lesson-7-launchedeffect-with-keys---dynamic-side-effects)
- [Lesson 8: Real-world Examples and Best Practices](#lesson-8-real-world-examples-and-best-practices)

---

## Lesson 1: Introduction to Side Effects

### What are Side Effects?

In Jetpack Compose, **side effects** are operations that occur outside the scope of composable functions and can affect the external state of your application. These are actions that are NOT part of the UI composition process but are necessary for the functionality of your app.

### Why Do We Need Side Effects?

Composable functions should be **pure functions** - they should only describe the UI based on their input parameters and not have side effects. However, real applications need to:
- Make network requests
- Update database
- Show toast messages
- Navigate between screens
- Start/stop animations
- Access device sensors
- Manage app state

These operations are **side effects** and need to be handled carefully in Compose.

### Types of Side Effects in Compose

Compose provides several APIs to handle different types of side effects:

1. **LaunchedEffect** - Executes a coroutine when the composable enters the composition
2. **DisposableEffect** - For side effects that need cleanup
3. **RememberCoroutineScope** - Creates a coroutine scope tied to composable lifecycle
4. **RememberUpdatedState** - Preserves the latest value for use in effects
5. **SideEffect** - Runs on every successful recomposition
6. **LaunchedEffect with Keys** - Re-executes when keys change

### The Problem with Direct Side Effects

❌ **Don't do this in composables:**
```kotlin
@Composable
fun BadExample() {
    val scope = rememberCoroutineScope()
    
    // This will cause infinite recompositions!
    scope.launch {
        while (true) {
            delay(1000)
            // This triggers state updates
            globalState.value = System.currentTimeMillis()
        }
    }
    
    Text("Time: ${System.currentTimeMillis()}")
}
```

**Problems:**
- Each recomposition starts a new coroutine
- Infinite loop causes memory leaks
- Performance degradation
- Unpredictable behavior

### The Compose Side Effects Solution

✅ **Use the appropriate side effect API:**
```kotlin
@Composable
fun GoodExample() {
    var time by remember { mutableStateOf(System.currentTimeMillis()) }
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            time = System.currentTimeMillis()
        }
    }
    
    Text("Time: $time")
}
```

**Benefits:**
- Single coroutine lifecycle
- Automatic cleanup
- Predictable behavior
- Better performance

### Key Principles

1. **Composables are declarative** - They describe WHAT the UI should look like
2. **Side effects are imperative** - They describe HOW to make things happen
3. **Side effects should be contained** - Use proper APIs to manage lifecycle
4. **Cleanup is crucial** - Always clean up resources when composable leaves composition

### When to Use Each Side Effect API

| API | Use Case | Lifecycle |
|-----|----------|-----------|
| `LaunchedEffect` | One-time operations, long-running tasks | Starts on composition, stops on disposal |
| `DisposableEffect` | Operations requiring cleanup | Same as LaunchedEffect, but with cleanup |
| `RememberCoroutineScope` | Manual coroutine management | Tied to composition lifecycle |
| `RememberUpdatedState` | Preserve latest values in effects | Updates without restarting effect |
| `SideEffect` | Debugging, logging, analytics | Runs on every recomposition |
| `LaunchedEffect(keys)` | Dynamic effects based on dependencies | Restarts when keys change |

### Setting Up the Environment

Before we dive into examples, let's set up a basic Compose project structure:

```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            SideEffectsTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colors.background
                ) {
                    SideEffectLessons()
                }
            }
        }
    }
}

@Composable
fun SideEffectLessons() {
    var currentLesson by remember { mutableStateOf(1) }
    
    Column {
        LazyColumn {
            items(1) { lesson ->
                when (lesson) {
                    1 -> LessonOneIntroduction()
                    // Add more lessons here
                }
            }
        }
    }
}
```

### Common Mistakes to Avoid

1. **Creating coroutines without proper scope**
2. **Not cleaning up resources**
3. **Using global state in composables**
4. **Triggering recompositions in side effects without proper keys**
5. **Mixing imperative code with declarative composables**

### Next Steps

In the next lesson, we'll dive deep into `LaunchedEffect` with practical examples and common patterns.

---

**Practice Exercise:**
Try to identify which of these operations would be considered side effects:
- Making a network request to fetch user data
- Calculating a Text size based on available width
- Saving data to SharedPreferences
- Displaying a dialog based on a boolean flag
- Starting a timer to update UI every second

Think about each operation and why it would or wouldn't be a side effect. We'll discuss the answers in the next lesson!

---

## Lesson 2: LaunchedEffect - One-time Side Effects

### What is LaunchedEffect?

`LaunchedEffect` is the most commonly used side effect API in Jetpack Compose. It launches a coroutine when the composable enters the composition and automatically cancels it when the composable leaves the composition.

### Basic Syntax

```kotlin
LaunchedEffect(key1, key2, ...) {
    // Coroutine code here
}
```

### Key Characteristics

1. **Runs in a coroutine** - Enables async operations
2. **Lifecycle managed by Compose** - Automatically started/stopped
3. **Re-execution based on keys** - Recreates when keys change
4. **No return value** - Used for effects, not for returning data

### Simple Example: Timer

Let's start with a simple timer that updates every second:

```kotlin
@Composable
fun SimpleTimer() {
    var time by remember { mutableStateOf(0L) }
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            time += 1
        }
    }
    
    Text("Time: $time seconds")
}
```

**Explanation:**
- `LaunchedEffect(Unit)` runs once when the composable is first composed
- The coroutine runs indefinitely, updating the time every second
- When the composable is removed, the coroutine is automatically cancelled

### Example 1: Loading Data with Network Request

```kotlin
data class User(val id: String, val name: String, val email: String)

@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }
    var isLoading by remember { mutableStateOf(true) }
    var error by remember { mutableStateOf<String?>(null) }
    
    // LaunchedEffect to fetch user data
    LaunchedEffect(userId) {
        isLoading = true
        error = null
        
        try {
            user = api.getUser(userId) // Simulated network call
        } catch (e: Exception) {
            error = e.message
        } finally {
            isLoading = false
        }
    }
    
    when {
        isLoading -> {
            CircularProgressIndicator()
        }
        error != null -> {
            Text("Error: $error")
        }
        user != null -> {
            UserDetails(user = user!!)
        }
    }
}

@Composable
fun UserDetails(user: User) {
    Column {
        Text("Name: ${user.name}")
        Text("Email: ${user.email}")
    }
}
```

### Example 2: Animating Values

```kotlin
@Composable
fun AnimatedCounter(targetValue: Int) {
    var currentValue by remember { mutableStateOf(0f) }
    
    LaunchedEffect(targetValue) {
        val startValue = currentValue
        val endValue = targetValue.toFloat()
        val duration = 1000L // 1 second
        
        val startTime = System.currentTimeMillis()
        
        while (currentValue != endValue) {
            val elapsed = System.currentTimeMillis() - startTime
            val progress = (elapsed.toFloat() / duration).coerceIn(0f, 1f)
            
            // Easing function for smooth animation
            currentValue = startValue + (endValue - startValue) * easeInOutCubic(progress)
            
            delay(16) // ~60fps
        }
        currentValue = endValue
    }
    
    Text("Value: ${currentValue.toInt()}")
}

private fun easeInOutCubic(t: Float): Float {
    return if (t < 0.5f) {
        4f * t * t * t
    } else {
        1f - (-2f * t + 2f).pow(3) / 2f
    }
}
```

### Example 3: Playing Sound Effects

```kotlin
@Composable
fun SoundButton(
    soundResource: Int,
    text: String,
    onClick: () -> Unit
) {
    var isEnabled by remember { mutableStateOf(true) }
    val context = LocalContext.current
    val soundPool = remember { SoundPool.Builder().setMaxStreams(1).build() }
    var soundId by remember { mutableStateOf(0) }
    
    // Load sound on composition
    LaunchedEffect(soundResource) {
        soundId = soundPool.load(context, soundResource, 1)
    }
    
    // Clean up sound resources
    DisposableEffect(Unit) {
        onDispose {
            soundPool.release()
        }
    }
    
    Button(
        onClick = {
            if (isEnabled) {
                soundPool.play(soundId, 1f, 1f, 0, 0, 1f)
                onClick()
            }
        },
        enabled = isEnabled
    ) {
        Text(text)
    }
}
```

### Example 4: Debounced Search

```kotlin
@Composable
fun DebouncedSearch(
    onSearch: (String) -> Unit
) {
    var query by remember { mutableStateOf("") }
    var isSearching by remember { mutableStateOf(false) }
    
    // Debounced search effect
    LaunchedEffect(query) {
        isSearching = true
        delay(500) // Wait 500ms after user stops typing
        
        if (query.isNotEmpty()) {
            onSearch(query)
        }
        isSearching = false
    }
    
    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it },
            label = { Text("Search") },
            trailingIcon = {
                if (isSearching) {
                    CircularProgressIndicator(modifier = Modifier.size(16.dp))
                }
            }
        )
    }
}
```

### Using LaunchedEffect with Keys

The power of `LaunchedEffect` comes from its key parameters. When any key changes, the effect is cancelled and restarted:

```kotlin
@Composable
fun UserProfileWithKeyExample(userId: String) {
    var userData by remember { mutableStateOf<User?>(null) }
    
    // This effect re-runs when userId changes
    LaunchedEffect(userId) {
        userData = fetchUserData(userId)
    }
    
    // This effect re-runs when BOTH userId and language change
    LaunchedEffect(userId, getCurrentLanguage()) {
        loadUserPreferences(userId, getCurrentLanguage())
    }
    
    userData?.let { user ->
        Text("Welcome, ${user.name}")
    }
}
```

### Common Patterns

#### Pattern 1: Loading with Retry

```kotlin
@Composable
fun RetryableNetworkCall(
    dataId: String,
    maxRetries: Int = 3
) {
    var data by remember { mutableStateOf<Data?>(null) }
    var error by remember { mutableStateOf<String?>(null) }
    var isLoading by remember { mutableStateOf(false) }
    var retryCount by remember { mutableStateOf(0) }
    
    LaunchedEffect(dataId, retryCount) {
        if (retryCount > maxRetries) {
            return@LaunchedEffect
        }
        
        isLoading = true
        error = null
        
        try {
            data = fetchDataWithRetry(dataId, retryCount)
            error = null // Success
        } catch (e: Exception) {
            error = "Failed: ${e.message}"
        } finally {
            isLoading = false
        }
    }
    
    when {
        isLoading -> LoadingState()
        error != null && retryCount <= maxRetries -> {
            ErrorState(
                error = error!!,
                onRetry = { retryCount++ }
            )
        }
        data != null -> DataState(data = data!!)
    }
}
```

#### Pattern 2: Periodic Data Refresh

```kotlin
@Composable
fun AutoRefreshingList(
    refreshInterval: Long = 30000L // 30 seconds
) {
    var items by remember { mutableStateOf<List<Item>>(emptyList()) }
    var isRefreshing by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        // Initial load
        refreshData()
        
        // Periodic refresh
        while (true) {
            delay(refreshInterval)
            isRefreshing = true
            refreshData()
            isRefreshing = false
        }
    }
    
    LaunchedEffect(isRefreshing) {
        // Only show loading indicator if not initial load
        if (isRefreshing) {
            refreshData()
        }
    }
    
    LazyColumn {
        items(items) { item ->
            ItemRow(item)
            if (isRefreshing && item == items.last()) {
                item {
                    LoadingIndicator()
                }
            }
        }
    }
}
```

### Best Practices

1. **Always use keys when the effect depends on external values**
2. **Keep the effect body simple and focused**
3. **Handle exceptions properly**
4. **Consider using `suspend` functions for complex operations**
5. **Use `remember` to store data that should persist across recompositions**

### Common Mistakes

❌ **Wrong: No keys for dependent values**
```kotlin
@Composable
fun BadExample(userId: String) {
    LaunchedEffect(Unit) {
        // This will use the initial userId forever
        val user = fetchUser(userId)
        updateUI(user)
    }
}
```

✅ **Correct: Use proper keys**
```kotlin
@Composable
fun GoodExample(userId: String) {
    LaunchedEffect(userId) {
        // This will re-run when userId changes
        val user = fetchUser(userId)
        updateUI(user)
    }
}
```

❌ **Wrong: Not handling cancellation**
```kotlin
@Composable
fun BadExample() {
    LaunchedEffect(Unit) {
        val result = longRunningOperation() // May not be cancelled
        updateState(result)
    }
}
```

✅ **Correct: Make operations cancellable**
```kotlin
@Composable
fun GoodExample() {
    LaunchedEffect(Unit) {
        val result = withContext(Dispatchers.IO) {
            longRunningOperation() // Will be cancelled with the coroutine
        }
        updateState(result)
    }
}
```

### Performance Considerations

1. **Avoid expensive operations in LaunchedEffect** - Move to background dispatchers
2. **Use appropriate keys** - Overuse of `Unit` key can cause unnecessary restarts
3. **Consider lazy loading** - Only start effects when needed
4. **Memory efficiency** - Clean up resources properly

### Advanced Example: Biometric Authentication

```kotlin
@Composable
fun BiometricAuthPrompt(
    onSuccess: () -> Unit,
    onError: (String) -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var isAuthenticating by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        val biometricManager = BiometricManager.from(context)
        
        when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_WEAK)) {
            BiometricManager.BIOMETRIC_SUCCESS -> {
                // Proceed with authentication
                isAuthenticating = true
                
                val promptInfo = BiometricPrompt.PromptInfo.Builder()
                    .setTitle("Biometric Authentication")
                    .setSubtitle("Use your fingerprint or face to authenticate")
                    .setNegativeButtonText("Cancel")
                    .build()
                
                val biometricPrompt = BiometricPrompt(
                    lifecycleOwner,
                    ContextCompat.getMainExecutor(context),
                    object : BiometricPrompt.AuthenticationCallback() {
                        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                            super.onAuthenticationSucceeded(result)
                            isAuthenticating = false
                            onSuccess()
                        }
                        
                        override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                            super.onAuthenticationError(errorCode, errString)
                            isAuthenticating = false
                            onError(errString.toString())
                        }
                    }
                )
                
                biometricPrompt.authenticate(promptInfo)
            }
            else -> {
                onError("Biometric authentication not available")
            }
        }
    }
    
    if (isAuthenticating) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center,
            modifier = Modifier.fillMaxSize()
        ) {
            CircularProgressIndicator()
            Spacer(modifier = Modifier.height(16.dp))
            Text("Authenticating...")
        }
    }
}
```

### Practice Exercise

Create a composable that:
1. Shows a "Start Task" button
2. When clicked, starts a 10-second countdown
3. Updates a progress bar during the countdown
4. Shows "Task Complete!" when finished
5. Can be reset to start again

**Hints:**
- Use `LaunchedEffect` for the countdown timer
- Use a state variable to track progress
- Consider using `LaunchedEffect` keys for restart functionality

**Answer:**
```kotlin
@Composable
fun TaskCountdown() {
    var isRunning by remember { mutableStateOf(false) }
    var progress by remember { mutableStateOf(0f) }
    var timeLeft by remember { mutableStateOf(10) }
    
    LaunchedEffect(isRunning, timeLeft) {
        if (isRunning && timeLeft > 0) {
            delay(1000)
            timeLeft -= 1
            progress = (10 - timeLeft) / 10f
            
            if (timeLeft == 0) {
                isRunning = false
            }
        }
    }
    
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
        modifier = Modifier.fillMaxSize()
    ) {
        if (timeLeft > 0) {
            Text("Time Left: $timeLeft seconds")
            Spacer(modifier = Modifier.height(16.dp))
            
            LinearProgressIndicator(
                progress = progress,
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            )
            
            Spacer(modifier = Modifier.height(16.dp))
            
            if (isRunning) {
                Text("Task in progress...")
            } else if (progress > 0) {
                Text("Task Complete!")
            }
        } else {
            Text("Task Complete!")
            Spacer(modifier = Modifier.height(16.dp))
            Text("Great job!")
        }
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Row {
            Button(
                onClick = { 
                    if (!isRunning && timeLeft == 0) {
                        // Reset
                        timeLeft = 10
                        progress = 0f
                    } else if (!isRunning) {
                        // Start
                        isRunning = true
                    }
                }
            ) {
                Text(
                    when {
                        timeLeft == 0 -> "Restart Task"
                        !isRunning -> "Start Task"
                        else -> "Task Running..."
                    }
                )
            }
            
            Spacer(modifier = Modifier.width(16.dp))
            
            if (isRunning) {
                Button(
                    onClick = {
                        isRunning = false
                        timeLeft = 10
                        progress = 0f
                    }
                ) {
                    Text("Reset")
                }
            }
        }
    }
}
```

### Key Takeaways

- `LaunchedEffect` is your go-to for one-time or long-running operations
- Always consider what values your effect depends on for proper key usage
- Handle cancellation and cleanup properly
- Use appropriate dispatchers for blocking operations
- The effect runs in a coroutine, so you can use all suspend functions

---

## Lesson 3: DisposableEffect - Cleanup Side Effects

### What is DisposableEffect?

`DisposableEffect` is similar to `LaunchedEffect` but with one crucial addition: it provides a cleanup function that's called when the composable leaves the composition or when any of the keys change. This is essential for managing resources that need proper cleanup.

### Basic Syntax

```kotlin
DisposableEffect(key1, key2, ...) {
    // Setup code here - runs when composable enters composition
    
    onDispose {
        // Cleanup code here - runs when composable leaves composition or keys change
    }
}
```

### Key Characteristics

1. **Setup and cleanup** - Both phases are clearly defined
2. **Lifecycle managed by Compose** - Automatic setup and cleanup
3. **Re-execution based on keys** - Recreates when keys change
4. **Resource management** - Perfect for objects that need cleanup
5. **Runs outside of coroutine scope** - No access to coroutine features

### When to Use DisposableEffect

Use `DisposableEffect` when you need to:
- **Clean up resources** (release memory, close connections)
- **Unregister listeners** (remove observers, event handlers)
- **Stop background operations** (cancel timers, stop sensors)
- **Release system resources** (release camera, audio, sensors)
- **Save state** (persist data before leaving)

### Simple Example: Timer with Cleanup

Let's start with a timer that properly cleans up:

```kotlin
@Composable
fun DisposableTimer() {
    var time by remember { mutableStateOf(0L) }
    var isRunning by remember { mutableStateOf(false) }
    
    DisposableEffect(isRunning) {
        var timerJob: Job? = null
        
        if (isRunning) {
            timerJob = CoroutineScope(Dispatchers.Main).launch {
                while (isRunning) {
                    delay(1000)
                    time += 1
                }
            }
        }
        
        // Cleanup: Cancel timer when composable leaves or isRunning changes
        onDispose {
            timerJob?.cancel()
        }
    }
    
    Column {
        Text("Time: $time seconds")
        Button(
            onClick = { isRunning = !isRunning }
        ) {
            Text(if (isRunning) "Stop" else "Start")
        }
    }
}
```

**Explanation:**
- When `isRunning` becomes `true`, a timer is started
- When `isRunning` becomes `false` OR the composable leaves, the timer is cancelled
- The `onDispose` block ensures no memory leaks

### Example 1: Managing Media Player

```kotlin
@Composable
fun MediaPlayerComposable(
    mediaUri: String,
    isPlaying: Boolean
) {
    var player by remember { mutableStateOf<MediaPlayer?>(null) }
    val context = LocalContext.current
    
    DisposableEffect(Unit) {
        // Setup: Create and prepare the player
        player = MediaPlayer().apply {
            setDataSource(context, Uri.parse(mediaUri))
            prepare()
        }
        
        // Cleanup: Release the player resources
        onDispose {
            player?.release()
            player = null
        }
    }
    
    // Start/stop playback based on isPlaying state
    DisposableEffect(isPlaying) {
        if (isPlaying) {
            player?.start()
        } else {
            player?.pause()
        }
        
        onDispose {
            // Ensure we pause when leaving or state changes
            player?.pause()
        }
    }
    
    Column {
        Text("Media Player")
        Text("Status: ${if (isPlaying) "Playing" else "Paused"}")
    }
}
```

### Example 2: Camera Preview

```kotlin
@Composable
fun CameraPreview(
    cameraSelector: CameraSelector
) {
    val context = LocalContext.current
    var previewView by remember { mutableStateOf<PreviewView?>(null) }
    var cameraControl by remember { mutableStateOf<CameraControl?>(null) }
    
    DisposableEffect(Unit) {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(context)
        
        val cameraProvider = cameraProviderFuture.get()
        val preview = Preview.Builder().build().apply {
            setSurfaceProvider { surfaceProvider ->
                previewView?.surfaceProvider = surfaceProvider
            }
        }
        
        val camera = cameraProvider.bindToLifecycle(
            context as LifecycleOwner,
            cameraSelector,
            preview
        )
        
        cameraControl = camera.cameraControl
        
        onDispose {
            cameraProvider.unbindAll()
            cameraControl = null
        }
    }
    
    AndroidView(
        factory = { context ->
            PreviewView(context).apply {
                previewView = this
                implementationMode = PreviewView.ImplementationMode.COMPATIBLE
                scaleType = PreviewView.ScaleType.FILL_CENTER
            }
        },
        modifier = Modifier
            .fillMaxWidth()
            .height(300.dp)
    )
}
```

### Example 3: Location Updates

```kotlin
data class LocationData(
    val latitude: Double,
    val longitude: Double,
    val accuracy: Float
)

@Composable
fun LocationTracker(
    isTracking: Boolean,
    onLocationUpdate: (LocationData) -> Unit
) {
    val context = LocalContext.current
    var locationManager by remember { mutableStateOf<LocationManager?>(null) }
    
    DisposableEffect(Unit) {
        locationManager = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
        
        onDispose {
            locationManager = null
        }
    }
    
    DisposableEffect(isTracking) {
        var locationListener: LocationListener? = null
        
        if (isTracking) {
            locationListener = object : LocationListener {
                override fun onLocationChanged(location: Location) {
                    onLocationUpdate(
                        LocationData(
                            latitude = location.latitude,
                            longitude = location.longitude,
                            accuracy = location.accuracy
                        )
                    )
                }
                
                override fun onProviderEnabled(provider: String) {}
                override fun onProviderDisabled(provider: String) {}
                override fun onStatusChanged(provider: String?, status: Int, extras: Bundle?) {}
            }
            
            // Request location updates
            locationManager?.requestLocationUpdates(
                LocationManager.GPS_PROVIDER,
                1000L, // 1 second
                0f,    // 0 meters
                locationListener
            )
        }
        
        onDispose {
            // Remove location updates
            locationListener?.let { listener ->
                locationManager?.removeUpdates(listener)
            }
        }
    }
    
    if (isTracking) {
        Text("Tracking location...")
    } else {
        Text("Location tracking stopped")
    }
}
```

### Example 4: Bluetooth LE Scanner

```kotlin
data class BluetoothDevice(val name: String, val address: String, val rssi: Int)

@Composable
fun BluetoothLEScanner(
    isScanning: Boolean,
    onDeviceFound: (BluetoothDevice) -> Unit
) {
    val context = LocalContext.current
    var bluetoothLeScanner by remember { mutableStateOf<BluetoothLeScanner?>(null) }
    var scanCallback by remember { mutableStateOf<BleScanCallback?>(null) }
    
    DisposableEffect(Unit) {
        val bluetoothManager = context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
        val bluetoothAdapter = bluetoothManager.adapter
        
        if (bluetoothAdapter != null && bluetoothAdapter.isEnabled) {
            bluetoothLeScanner = bluetoothAdapter.bluetoothLeScanner
        }
        
        onDispose {
            bluetoothLeScanner = null
        }
    }
    
    DisposableEffect(isScanning) {
        var scanningJob: Job? = null
        
        if (isScanning && bluetoothLeScanner != null) {
            scanCallback = BleScanCallback { device, rssi ->
                onDeviceFound(
                    BluetoothDevice(
                        name = device.name ?: "Unknown",
                        address = device.address,
                        rssi = rssi
                    )
                )
            }
            
            // Start scanning
            bluetoothLeScanner?.startScan(
                listOf(ScanFilter.Builder().build()),
                ScanSettings.Builder().build(),
                scanCallback!!
            )
            
            // Stop after 10 seconds
            scanningJob = CoroutineScope(Dispatchers.Main).launch {
                delay(10000)
                bluetoothLeScanner?.stopScan(scanCallback!!)
            }
        }
        
        onDispose {
            scanningJob?.cancel()
            
            // Stop scanning
            scanCallback?.let { callback ->
                bluetoothLeScanner?.stopScan(callback)
            }
            scanCallback = null
        }
    }
    
    if (isScanning) {
        Text("Scanning for Bluetooth devices...")
    } else {
        Text("Bluetooth scanning stopped")
    }
}

private class BleScanCallback(
    private val onDeviceFound: (BluetoothDevice, Int) -> Unit
) : ScanCallback() {
    override fun onScanResult(callbackType: Int, result: ScanResult) {
        super.onScanResult(callbackType, result)
        onDeviceFound(result.device, result.rssi)
    }
}
```

### Example 5: File Observer for Directory Monitoring

```kotlin
@Composable
fun DirectoryWatcher(
    directoryPath: String,
    isWatching: Boolean,
    onFileEvent: (String, Int) -> Unit // filename, event type
) {
    val context = LocalContext.current
    var fileObserver by remember { mutableStateOf<FileObserver?>(null) }
    
    DisposableEffect(Unit) {
        val directory = File(directoryPath)
        if (directory.exists() && directory.isDirectory) {
            fileObserver = object : FileObserver(directoryPath) {
                override fun onEvent(event: Int, path: String?) {
                    if (path != null) {
                        onFileEvent(path, event)
                    }
                }
            }
        }
        
        onDispose {
            fileObserver?.stopWatching()
            fileObserver = null
        }
    }
    
    DisposableEffect(isWatching) {
        if (isWatching && fileObserver != null) {
            fileObserver?.startWatching()
        } else {
            fileObserver?.stopWatching()
        }
        
        onDispose {
            fileObserver?.stopWatching()
        }
    }
    
    Text(
        "Watching: $directoryPath - " +
        if (isWatching) "Active" else "Inactive"
    )
}
```

### Example 6: Network Connectivity Monitor

```kotlin
@Composable
fun NetworkConnectivityMonitor(
    onConnectivityChanged: (Boolean) -> Unit
) {
    val context = LocalContext.current
    var connectivityManager by remember { mutableStateOf<ConnectivityManager?>(null) }
    var networkCallback by remember { mutableStateOf<ConnectivityManager.NetworkCallback?>(null) }
    
    DisposableEffect(Unit) {
        connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        
        onDispose {
            connectivityManager = null
        }
    }
    
    DisposableEffect(Unit) {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                onConnectivityChanged(true)
            }
            
            override fun onLost(network: Network) {
                onConnectivityChanged(false)
            }
        }
        
        networkCallback = callback
        
        // Register callback
        connectivityManager?.registerDefaultNetworkCallback(callback)
        
        onDispose {
            // Unregister callback
            networkCallback?.let { cb ->
                connectivityManager?.unregisterNetworkCallback(cb)
            }
            networkCallback = null
        }
    }
    
    Text("Network monitoring active")
}
```

### Common Patterns

#### Pattern 1: Resource Acquisition with Proper Cleanup

```kotlin
@Composable
fun ResourceManager<AutoCloseable>(
    createResource: () -> AutoCloseable,
    resourceKey: Any,
    useResource: (AutoCloseable) -> Unit
) {
    var resource by remember { mutableStateOf<AutoCloseable?>(null) }
    
    DisposableEffect(resourceKey) {
        val newResource = createResource()
        resource = newResource
        
        // Use the resource
        useResource(newResource)
        
        onDispose {
            // Cleanup: Close the resource
            newResource.close()
            resource = null
        }
    }
}
```

#### Pattern 2: Lifecycle-Aware Operations

```kotlin
@Composable
fun LifecycleAwareSensor(
    sensorType: Int,
    isActive: Boolean
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var sensorManager by remember { mutableStateOf<SensorManager?>(null) }
    var sensor by remember { mutableStateOf<Sensor?>(null) }
    var sensorEventListener by remember { mutableStateOf<SensorEventListener?>(null) }
    
    DisposableEffect(Unit) {
        sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        sensor = sensorManager?.getDefaultSensor(sensorType)
        
        onDispose {
            sensorManager = null
            sensor = null
        }
    }
    
    DisposableEffect(isActive, sensor) {
        var sensorJob: Job? = null
        
        if (isActive && sensor != null && sensorManager != null) {
            val listener = object : SensorEventListener {
                override fun onSensorChanged(event: SensorEvent) {
                    // Handle sensor data
                    Log.d("Sensor", "Value: ${event.values[0]}")
                }
                
                override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) {
                    // Handle accuracy changes
                }
            }
            
            sensorEventListener = listener
            
            // Register sensor
            sensorManager?.registerListener(
                listener,
                sensor,
                SensorManager.SENSOR_DELAY_NORMAL
            )
        }
        
        onDispose {
            sensorJob?.cancel()
            
            // Unregister sensor
            sensorEventListener?.let { listener ->
                sensorManager?.unregisterListener(listener)
            }
            sensorEventListener = null
        }
    }
    
    Text("Sensor: ${sensor?.name ?: "None"}")
}
```

### Best Practices

1. **Always cleanup resources** - Prevent memory leaks and resource exhaustion
2. **Use specific keys** - Only recreate when dependencies actually change
3. **Handle exceptions in cleanup** - Ensure cleanup runs even if it throws
4. **Keep cleanup idempotent** - Cleanup should be safe to call multiple times
5. **Avoid blocking in cleanup** - Cleanup should be quick and responsive

### Common Mistakes

❌ **Wrong: No cleanup**
```kotlin
@Composable
fun BadExample() {
    LaunchedEffect(Unit) {
        val mediaPlayer = MediaPlayer()
        mediaPlayer.setDataSource("...")
        mediaPlayer.prepare()
        // No cleanup - memory leak!
    }
}
```

✅ **Correct: Proper cleanup**
```kotlin
@Composable
fun GoodExample() {
    var player by remember { mutableStateOf<MediaPlayer?>(null) }
    
    DisposableEffect(Unit) {
        val mediaPlayer = MediaPlayer()
        mediaPlayer.setDataSource("...")
        mediaPlayer.prepare()
        player = mediaPlayer
        
        onDispose {
            mediaPlayer.release()
            player = null
        }
    }
}
```

❌ **Wrong: Cleanup in wrong place**
```kotlin
@Composable
fun BadExample(isActive: Boolean) {
    DisposableEffect(Unit) {
        if (isActive) {
            startTracking()
        }
        // This cleanup runs even when isActive changes, but tracking might still be active
        onDispose {
            stopTracking() // Wrong - should only cleanup when truly done
        }
    }
}
```

✅ **Correct: Key-based cleanup**
```kotlin
@Composable
fun GoodExample(isActive: Boolean) {
    DisposableEffect(isActive) {
        if (isActive) {
            startTracking()
        } else {
            stopTracking()
        }
        
        onDispose {
            // Only cleanup when composable is removed
            if (isActive) {
                stopTracking()
            }
        }
    }
}
```

### Performance Considerations

1. **Resource pooling** - Reuse expensive resources when possible
2. **Lazy initialization** - Only create resources when needed
3. **Background cleanup** - Move heavy cleanup to background thread if necessary
4. **Monitoring** - Track resource usage to detect leaks

### Advanced Example: WebView with Cleanup

```kotlin
@Composable
fun WebViewWithCleanup(
    url: String,
    onPageFinished: (String) -> Unit
) {
    var webView by remember { mutableStateOf<WebView?>(null) }
    val context = LocalContext.current
    var webViewClient by remember { mutableStateOf<WebViewClient?>(null) }
    
    DisposableEffect(Unit) {
        val client = object : WebViewClient() {
            override fun onPageFinished(view: WebView?, url: String?) {
                super.onPageFinished(view, url)
                url?.let { onPageFinished(it) }
            }
        }
        
        webViewClient = client
        
        onDispose {
            webViewClient = null
        }
    }
    
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webView = this
                settings.javaScriptEnabled = true
                webViewClient = webViewClient
                loadUrl(url)
            }
        },
        modifier = Modifier.fillMaxSize()
    )
    
    DisposableEffect(Unit) {
        onDispose {
            // Clean up WebView
            webView?.destroy()
            webView = null
        }
    }
}
```

### Practice Exercise

Create a composable that:
1. Takes a list of URLs to monitor
2. Starts a coroutine that checks each URL every 30 seconds
3. Tracks which URLs are responding and which are failing
4. Updates a list showing the status of each URL
5. Properly cleans up all monitoring when the composable is removed

**Hints:**
- Use `DisposableEffect` for cleanup of monitoring coroutines
- Use `LaunchedEffect` for the periodic checking logic
- Store the monitoring state in remember variables
- Ensure all coroutines are properly cancelled

### Key Takeaways

- `DisposableEffect` is essential for any resource that needs cleanup
- Always use `onDispose` to release resources and prevent memory leaks
- Use keys to control when effects are recreated and cleaned up
- Cleanup should be quick, safe, and idempotent
- Common resources needing cleanup: MediaPlayer, Camera, Sensors, Listeners, Files
- Combine `DisposableEffect` with `LaunchedEffect` for complex scenarios

---

## Lesson 4: RememberCoroutineScope - Coroutine Scope Management

### What is RememberCoroutineScope?

`RememberCoroutineScope` is a side effect that creates a coroutine scope tied to the lifecycle of a composable. Unlike `LaunchedEffect` which automatically starts a coroutine, `RememberCoroutineScope` gives you manual control over when to launch coroutines while still ensuring proper cleanup.

### Basic Syntax

```kotlin
val scope = rememberCoroutineScope()

// Later in the composable
scope.launch {
    // Your coroutine code
}
```

### Key Characteristics

1. **Manual coroutine control** - You decide when to launch coroutines
2. **Lifecycle tied to composable** - Automatically cancelled when composable leaves
3. **No auto-start** - Unlike `LaunchedEffect`, doesn't run automatically
4. **Flexible usage** - Can launch coroutines based on user interactions
5. **Composable-scoped** - Uses the composable's lifecycle for cancellation

### When to Use RememberCoroutineScope

Use `RememberCoroutineScope` when you need to:
- **Launch coroutines based on user events** (button clicks, form submissions)
- **Manual control over coroutine timing** (not automatic on composition)
- **Multiple coroutines with different triggers** (different from single LaunchedEffect)
- **Cancel coroutines separately** (different cancellation strategies)
- **Integration with event handlers** (callbacks, listeners)

### Simple Example: User-Triggered Action

Let's start with a simple example where a coroutine is launched when a button is clicked:

```kotlin
@Composable
fun UserTriggeredTask() {
    var isProcessing by remember { mutableStateOf(false) }
    var result by remember { mutableStateOf<String?>(null) }
    val scope = rememberCoroutineScope()
    
    Column {
        Button(
            onClick = {
                if (!isProcessing) {
                    isProcessing = true
                    result = null
                    
                    // Launch coroutine manually
                    scope.launch {
                        val taskResult = performBackgroundTask()
                        result = taskResult
                        isProcessing = false
                    }
                }
            },
            enabled = !isProcessing
        ) {
            Text(if (isProcessing) "Processing..." else "Start Task")
        }
        
        result?.let { resultText ->
            Text("Result: $resultText")
        }
    }
}

private suspend fun performBackgroundTask(): String {
    delay(2000) // Simulate work
    return "Task completed successfully"
}
```

**Explanation:**
- The coroutine is only launched when the button is clicked
- The scope is created once and persists across recompositions
- When the composable leaves, the scope is automatically cancelled
- Multiple clicks can be handled by checking `isProcessing` state

### Example 1: File Download with Progress

```kotlin
@Composable
fun FileDownloader(
    downloadUrl: String,
    fileName: String
) {
    var isDownloading by remember { mutableStateOf(false) }
    var progress by remember { mutableStateOf(0f) }
    var downloadStatus by remember { mutableStateOf<DownloadStatus?>(null) }
    val scope = rememberCoroutineScope()
    
    DisposableEffect(Unit) {
        onDispose {
            // Clean up any ongoing downloads if needed
            if (isDownloading) {
                // Could implement cancel logic here
            }
        }
    }
    
    Column {
        Text("Download: $fileName")
        
        if (isDownloading) {
            LinearProgressIndicator(
                progress = progress,
                modifier = Modifier.fillMaxWidth()
            )
            Text("${(progress * 100).toInt()}%")
        }
        
        downloadStatus?.let { status ->
            Text("Status: $status")
        }
        
        Row {
            Button(
                onClick = {
                    if (!isDownloading) {
                        isDownloading = true
                        progress = 0f
                        downloadStatus = DownloadStatus.DOWNLOADING
                        
                        scope.launch {
                            val file = downloadFile(downloadUrl, fileName) { currentProgress ->
                                progress = currentProgress
                            }
                            
                            downloadStatus = if (file != null) {
                                DownloadStatus.COMPLETED
                            } else {
                                DownloadStatus.FAILED
                            }
                            isDownloading = false
                        }
                    }
                },
                enabled = !isDownloading
            ) {
                Text("Download")
            }
            
            if (isDownloading) {
                Button(
                    onClick = {
                        scope.coroutineContext.job.cancel()
                        isDownloading = false
                        downloadStatus = DownloadStatus.CANCELLED
                    }
                ) {
                    Text("Cancel")
                }
            }
        }
    }
}

private suspend fun downloadFile(
    url: String,
    fileName: String,
    onProgress: (Float) -> Unit
): File? {
    return withContext(Dispatchers.IO) {
        try {
            // Simulate file download with progress
            repeat(100) { i ->
                delay(50) // Simulate network delay
                onProgress((i + 1) / 100f)
            }
            
            // Return a dummy file
            File("dummy_path")
        } catch (e: CancellationException) {
            // Handle cancellation
            null
        } catch (e: Exception) {
            null
        }
    }
}

enum class DownloadStatus {
    DOWNLOADING,
    COMPLETED,
    FAILED,
    CANCELLED
}
```

### Example 2: Search with Real-time Suggestions

```kotlin
@Composable
fun SearchWithSuggestions(
    searchFunction: suspend (String) -> List<String>
) {
    var query by remember { mutableStateOf("") }
    var suggestions by remember { mutableStateOf<List<String>>(emptyList()) }
    var isSearching by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()
    
    // State for managing search job
    var currentSearchJob by remember { mutableStateOf<Job?>(null) }
    
    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { newQuery ->
                query = newQuery
                
                // Cancel previous search if running
                currentSearchJob?.cancel()
                
                if (newQuery.isEmpty()) {
                    suggestions = emptyList()
                    isSearching = false
                    return@onValueChange
                }
                
                isSearching = true
                
                // Launch new search
                currentSearchJob = scope.launch {
                    delay(300) // Debounce
                    
                    try {
                        val results = withContext(Dispatchers.IO) {
                            searchFunction(newQuery)
                        }
                        suggestions = results
                    } catch (e: CancellationException) {
                        // Search was cancelled, ignore
                    } finally {
                        isSearching = false
                    }
                }
            },
            label = { Text("Search") },
            trailingIcon = {
                if (isSearching) {
                    CircularProgressIndicator(modifier = Modifier.size(16.dp))
                }
            }
        )
        
        if (suggestions.isNotEmpty()) {
            LazyColumn {
                items(suggestions) { suggestion ->
                    Text(
                        text = suggestion,
                        modifier = Modifier
                            .fillMaxWidth()
                            .clickable {
                                query = suggestion
                                suggestions = emptyList()
                            }
                            .padding(16.dp)
                    )
                }
            }
        }
    }
}
```

### Example 3: Multi-step Form Submission

```kotlin
data class FormStep(
    val title: String,
    val isCompleted: Boolean,
    val data: Any? = null
)

@Composable
fun MultiStepForm(
    steps: List<FormStep>
) {
    var currentStep by remember { mutableStateOf(0) }
    var formData by remember { mutableStateOf(mutableListOf<Any?>()) }
    var isSubmitting by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()
    
    DisposableEffect(Unit) {
        onDispose {
            // Clean up any ongoing submission
            if (isSubmitting) {
                // Could implement submission cancellation
            }
        }
    }
    
    Column {
        // Step indicator
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            steps.forEachIndexed { index, step ->
                Text(
                    text = "${index + 1}",
                    color = if (index <= currentStep) {
                        Color.Blue
                    } else {
                        Color.Gray
                    }
                )
                if (index < steps.size - 1) {
                    Spacer(modifier = Modifier.width(8.dp))
                    Divider(
                        modifier = Modifier
                            .width(20.dp)
                            .align(Alignment.CenterVertically),
                        color = if (index < currentStep) Color.Blue else Color.Gray
                    )
                }
            }
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Current step content
        Text(
            text = "Step ${currentStep + 1}: ${steps[currentStep].title}",
            style = MaterialTheme.typography.h6
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Form step specific content
        when (currentStep) {
            0 -> {
                var name by remember { mutableStateOf(formData.getOrNull(0) as? String ?: "") }
                
                OutlinedTextField(
                    value = name,
                    onValueChange = { name = it },
                    label = { Text("Full Name") }
                )
                
                LaunchedEffect(name) {
                    if (formData.size > 0) {
                        formData[0] = name
                    } else {
                        formData.add(name)
                    }
                }
            }
            1 -> {
                var email by remember { mutableStateOf(formData.getOrNull(1) as? String ?: "") }
                
                OutlinedTextField(
                    value = email,
                    onValueChange = { email = it },
                    label = { Text("Email") }
                )
                
                LaunchedEffect(email) {
                    if (formData.size > 1) {
                        formData[1] = email
                    } else {
                        formData.add(email)
                    }
                }
            }
            // Add more steps as needed
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Navigation buttons
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            if (currentStep > 0) {
                Button(
                    onClick = { currentStep-- }
                ) {
                    Text("Previous")
                }
            }
            
            if (currentStep < steps.size - 1) {
                Button(
                    onClick = {
                        // Validate current step
                        if (validateCurrentStep(currentStep, formData)) {
                            currentStep++
                        }
                    },
                    enabled = !isSubmitting
                ) {
                    Text("Next")
                }
            } else {
                Button(
                    onClick = {
                        isSubmitting = true
                        
                        scope.launch {
                            val result = submitForm(formData)
                            
                            if (result.isSuccess) {
                                // Handle success
                                showSuccessMessage()
                            } else {
                                // Handle failure
                                showErrorMessage(result.exception?.message)
                            }
                            
                            isSubmitting = false
                        }
                    },
                    enabled = !isSubmitting
                ) {
                    if (isSubmitting) {
                        Row {
                            CircularProgressIndicator(
                                modifier = Modifier.size(16.dp),
                                color = Color.White
                            )
                            Spacer(modifier = Modifier.width(8.dp))
                            Text("Submitting...")
                        }
                    } else {
                        Text("Submit")
                    }
                }
            }
        }
    }
}

private suspend fun validateCurrentStep(step: Int, data: MutableList<Any?>): Boolean {
    delay(500) // Simulate validation
    return when (step) {
        0 -> (data.getOrNull(0) as? String)?.isNotEmpty() == true
        1 -> (data.getOrNull(1) as? String)?.contains("@") == true
        else -> true
    }
}

private suspend fun submitForm(data: MutableList<Any?>): Result<Unit> {
    return withContext(Dispatchers.IO) {
        try {
            delay(2000) // Simulate API call
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### Example 4: Event-Driven Data Synchronization

```kotlin
data class SyncEvent(
    val type: SyncType,
    val timestamp: Long
)

enum class SyncType {
    USER_LOGIN,
    DATA_CHANGE,
    TIME_BASED,
    MANUAL
}

@Composable
fun DataSyncManager(
    isUserLoggedIn: Boolean,
    onSyncTriggered: () -> Unit
) {
    var lastSyncTime by remember { mutableStateOf(0L) }
    var pendingEvents by remember { mutableStateOf(mutableListOf<SyncEvent>()) }
    var isSyncing by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()
    
    // Auto-sync based on user state
    LaunchedEffect(isUserLoggedIn) {
        if (isUserLoggedIn) {
            val event = SyncEvent(SyncType.USER_LOGIN, System.currentTimeMillis())
            pendingEvents.add(event)
        }
    }
    
    // Time-based sync
    LaunchedEffect(Unit) {
        while (true) {
            delay(5 * 60 * 1000) // 5 minutes
            
            if (isUserLoggedIn) {
                val event = SyncEvent(SyncType.TIME_BASED, System.currentTimeMillis())
                pendingEvents.add(event)
                triggerSync()
            }
        }
    }
    
    Column {
        Text("Data Sync Manager")
        Text("Pending events: ${pendingEvents.size}")
        Text("Last sync: ${SimpleDateFormat("HH:mm:ss").format(Date(lastSyncTime))}")
        
        Button(
            onClick = { 
                val event = SyncEvent(SyncType.MANUAL, System.currentTimeMillis())
                pendingEvents.add(event)
                triggerSync()
            }
        ) {
            Text("Manual Sync")
        }
        
        if (isSyncing) {
            Text("Syncing...")
        }
    }
    
    // Function to trigger sync
    fun triggerSync() {
        scope.launch {
            if (isSyncing) return@launch
            
            isSyncing = true
            
            try {
                val eventsToProcess = pendingEvents.toList()
                pendingEvents.clear()
                
                withContext(Dispatchers.IO) {
                    // Process sync events
                    processSyncEvents(eventsToProcess)
                }
                
                lastSyncTime = System.currentTimeMillis()
                onSyncTriggered()
                
            } catch (e: Exception) {
                // Return events to queue on failure
                pendingEvents.addAll(eventsToProcess)
            } finally {
                isSyncing = false
            }
        }
    }
}

private suspend fun processSyncEvents(events: List<SyncEvent>) {
    events.forEach { event ->
        delay(200) // Simulate processing time
        
        when (event.type) {
            SyncType.USER_LOGIN -> handleUserLoginSync()
            SyncType.DATA_CHANGE -> handleDataChangeSync()
            SyncType.TIME_BASED -> handleTimeBasedSync()
            SyncType.MANUAL -> handleManualSync()
        }
    }
}
```

### Example 5: Complex Animation Controller

```kotlin
@Composable
fun AnimationController(
    animationType: AnimationType
) {
    var isAnimating by remember { mutableStateOf(false) }
    var progress by remember { mutableStateOf(0f) }
    var isPaused by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()
    
    // Current animation job
    var currentAnimationJob by remember { mutableStateOf<Job?>(null) }
    
    DisposableEffect(Unit) {
        onDispose {
            currentAnimationJob?.cancel()
        }
    }
    
    Column {
        // Animation display area
        Box(
            modifier = Modifier
                .size(200.dp)
                .background(
                    color = Color.Blue.copy(alpha = progress),
                    shape = if (animationType == AnimationType.CIRCLE) {
                        CircleShape
                    } else {
                        RoundedCornerShape(0.dp)
                    }
                )
        ) {
            Text(
                text = "${(progress * 100).toInt()}%",
                color = Color.White,
                modifier = Modifier.align(Alignment.Center)
            )
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // Animation controls
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            Button(
                onClick = {
                    startAnimation()
                },
                enabled = !isAnimating
            ) {
                Text("Start")
            }
            
            Button(
                onClick = {
                    if (isAnimating) {
                        if (isPaused) {
                            resumeAnimation()
                        } else {
                            pauseAnimation()
                        }
                    }
                },
                enabled = isAnimating
            ) {
                Text(if (isPaused) "Resume" else "Pause")
            }
            
            Button(
                onClick = {
                    stopAnimation()
                },
                enabled = isAnimating
            ) {
                Text("Stop")
            }
        }
        
        if (isAnimating) {
            Text("Animation: ${animationType.name}")
            Text("Status: ${if (isPaused) "Paused" else "Running"}")
        }
    }
    
    fun startAnimation() {
        // Cancel any existing animation
        currentAnimationJob?.cancel()
        
        isAnimating = true
        isPaused = false
        progress = 0f
        
        currentAnimationJob = scope.launch {
            try {
                val duration = when (animationType) {
                    AnimationType.FAST -> 1000L
                    AnimationType.MEDIUM -> 3000L
                    AnimationType.SLOW -> 5000L
                }
                
                val startTime = System.currentTimeMillis()
                
                while (progress < 1f && isActive) {
                    if (!isPaused) {
                        val elapsed = System.currentTimeMillis() - startTime
                        progress = (elapsed.toFloat() / duration).coerceAtMost(1f)
                    }
                    delay(16) // ~60fps
                }
                
                progress = 1f
            } finally {
                isAnimating = false
            }
        }
    }
    
    fun pauseAnimation() {
        isPaused = true
    }
    
    fun resumeAnimation() {
        isPaused = false
    }
    
    fun stopAnimation() {
        currentAnimationJob?.cancel()
        isAnimating = false
        isPaused = false
        progress = 0f
    }
}

enum class AnimationType {
    FAST,
    MEDIUM,
    SLOW,
    CIRCLE
}
```

### Common Patterns

#### Pattern 1: Event-Based Coroutine Launching

```kotlin
@Composable
fun EventBasedHandler(
    events: Flow<UIEvent>
) {
    val scope = rememberCoroutineScope()
    
    LaunchedEffect(Unit) {
        events.collect { event ->
            when (event) {
                is UIEvent.ShowMessage -> {
                    scope.launch {
                        showMessage(event.message)
                    }
                }
                is UIEvent.NavigateTo -> {
                    scope.launch {
                        navigateTo(event.destination)
                    }
                }
                is UIEvent.SaveData -> {
                    scope.launch {
                        saveData(event.data)
                    }
                }
            }
        }
    }
}
```

#### Pattern 2: Request-Cancellation Pattern

```kotlin
@Composable
fun RequestManager<T>(
    request: T,
    executeRequest: suspend (T) -> Result<Any>
) {
    var isLoading by remember { mutableStateOf(false) }
    var result by remember { mutableStateOf<Result<Any>?>(null) }
    val scope = rememberCoroutineScope()
    
    var currentRequestId by remember { mutableStateOf(0L) }
    
    DisposableEffect(request) {
        val requestId = currentRequestId + 1
        currentRequestId = requestId
        
        isLoading = true
        result = null
        
        val job = scope.launch {
            val requestResult = withContext(Dispatchers.IO) {
                executeRequest(request)
            }
            
            // Only update if this is still the latest request
            if (requestId == currentRequestId) {
                result = requestResult
                isLoading = false
            }
        }
        
        onDispose {
            job.cancel()
        }
    }
}
```

### Best Practices

1. **Always use scope for user-initiated actions** - Button clicks, form submissions, etc.
2. **Manage coroutine cancellation properly** - Cancel when composable leaves
3. **Use ` DisposableEffect` for scope cleanup** - Ensure all coroutines are cancelled
4. **Handle exceptions in coroutines** - Prevent crashes from unhandled exceptions
5. **Use appropriate dispatchers** - IO for blocking work, Main for UI updates
6. **Consider debouncing for frequent events** - Search, typing, etc.

### Common Mistakes

❌ **Wrong: Using Global Scope**
```kotlin
@Composable
fun BadExample() {
    Button(onClick = {
        // This creates a coroutine that outlives the composable!
        GlobalScope.launch {
            performTask()
        }
    }) {
        Text("Click me")
    }
}
```

✅ **Correct: Using RememberCoroutineScope**
```kotlin
@Composable
fun GoodExample() {
    val scope = rememberCoroutineScope()
    
    Button(onClick = {
        scope.launch {
            performTask()
        }
    }) {
        Text("Click me")
    }
}
```

❌ **Wrong: Multiple scopes without coordination**
```kotlin
@Composable
fun BadExample() {
    val scope1 = rememberCoroutineScope()
    val scope2 = rememberCoroutineScope()
    
    // Different scopes make coordination difficult
    Button(onClick = {
        scope1.launch { task1() }
        scope2.launch { task2() }
    }) {
        Text("Click me")
    }
}
```

✅ **Correct: Single scope for coordinated tasks**
```kotlin
@Composable
fun GoodExample() {
    val scope = rememberCoroutineScope()
    
    Button(onClick = {
        scope.launch {
            task1()
            task2() // Runs after task1
        }
    }) {
        Text("Click me")
    }
}
```

### Performance Considerations

1. **Scope reuse** - Don't create new scopes unnecessarily
2. **Job cancellation** - Cancel jobs when no longer needed
3. **Memory efficiency** - Avoid capturing large objects in closures
4. **Background work** - Always use appropriate dispatchers

### Advanced Example: Real-time Chat Manager

```kotlin
data class ChatMessage(
    val id: String,
    val userId: String,
    val content: String,
    val timestamp: Long,
    val isRead: Boolean = false
)

@Composable
fun RealTimeChatManager(
    roomId: String,
    isConnected: Boolean,
    onMessageReceived: (ChatMessage) -> Unit
) {
    var messages by remember { mutableStateOf<List<ChatMessage>>(emptyList()) }
    var connectionStatus by remember { mutableStateOf(ConnectionStatus.DISCONNECTED) }
    var typingUsers by remember { mutableStateOf(setOf<String>()) }
    val scope = rememberCoroutineScope()
    
    // Connection management
    DisposableEffect(Unit) {
        if (isConnected) {
            connectToChat()
        }
        
        onDispose {
            disconnectFromChat()
        }
    }
    
    // Auto-reconnect logic
    LaunchedEffect(connectionStatus) {
        if (connectionStatus == ConnectionStatus.FAILED && isConnected) {
            scope.launch {
                delay(5000) // Wait 5 seconds before retry
                if (isConnected) {
                    connectToChat()
                }
            }
        }
    }
    
    Column {
        // Connection status indicator
        Row(
            verticalAlignment = Alignment.CenterVertically
        ) {
            CircleIndicator(
                color = when (connectionStatus) {
                    ConnectionStatus.CONNECTED -> Color.Green
                    ConnectionStatus.CONNECTING -> Color.Yellow
                    ConnectionStatus.FAILED -> Color.Red
                    else -> Color.Gray
                }
            )
            Spacer(modifier = Modifier.width(8.dp))
            Text("Status: ${connectionStatus.name}")
        }
        
        // Messages list
        LazyColumn(
            modifier = Modifier
                .weight(1f)
                .fillMaxWidth()
        ) {
            items(messages) { message ->
                ChatMessageItem(message)
            }
        }
        
        // Typing indicator
        if (typingUsers.isNotEmpty()) {
            Text("Users typing: ${typingUsers.joinToString()}")
        }
        
        // Chat input (would be in separate composable)
        ChatInput(
            onSendMessage = { content ->
                scope.launch {
                    sendMessage(content)
                }
            },
            onTyping = { isTyping ->
                scope.launch {
                    sendTypingStatus(isTyping)
                }
            }
        )
    }
    
    private suspend fun connectToChat() {
        connectionStatus = ConnectionStatus.CONNECTING
        
        try {
            withContext(Dispatchers.IO) {
                // Simulate connection logic
                delay(1000)
                // connectToRoom(roomId)
            }
            
            connectionStatus = ConnectionStatus.CONNECTED
            
            // Start listening for messages
            scope.launch {
                listenForMessages(roomId) { message ->
                    messages = messages + message
                    onMessageReceived(message)
                }
            }
            
            // Start listening for typing status
            scope.launch {
                listenForTypingStatus(roomId) { userId, isTyping ->
                    if (isTyping) {
                        typingUsers = typingUsers + userId
                    } else {
                        typingUsers = typingUsers - userId
                    }
                }
            }
            
        } catch (e: Exception) {
            connectionStatus = ConnectionStatus.FAILED
        }
    }
    
    private suspend fun disconnectFromChat() {
        connectionStatus = ConnectionStatus.DISCONNECTED
        // disconnectFromRoom()
    }
}

private suspend fun listenForMessages(
    roomId: String,
    onMessage: (ChatMessage) -> Unit
) {
    // Simulate real-time message listening
    while (true) {
        delay(1000)
        val message = ChatMessage(
            id = "msg_${System.currentTimeMillis()}",
            userId = "user123",
            content = "New message",
            timestamp = System.currentTimeMillis()
        )
        onMessage(message)
    }
}

private suspend fun listenForTypingStatus(
    roomId: String,
    onTypingChange: (String, Boolean) -> Unit
) {
    // Simulate typing status listening
    delay(2000)
    onTypingChange("user456", true)
    delay(3000)
    onTypingChange("user456", false)
}

enum class ConnectionStatus {
    CONNECTED,
    CONNECTING,
    DISCONNECTED,
    FAILED
}
```

### Practice Exercise

Create a composable that:
1. Manages a shopping cart
2. Allows users to add/remove items
3. Automatically saves cart to local storage every 5 seconds
4. Shows unsaved changes indicator
5. Manually saves cart when explicitly requested
6. Loads cart data when composable is first shown

**Requirements:**
- Use `RememberCoroutineScope` for manual coroutine control
- Use `DisposableEffect` for cleanup
- Use `LaunchedEffect` for initial loading
- Handle auto-save with debouncing
- Show loading states properly

### Key Takeaways

- `RememberCoroutineScope` provides manual control over coroutine execution
- Perfect for user-initiated actions and event-driven scenarios
- Always combine with `DisposableEffect` for proper cleanup
- Use when you need different coroutine triggers than automatic composition
- Common use cases: button clicks, form submissions, file operations, real-time updates
- Can coordinate multiple related coroutines within the same scope
- Provides fine-grained control over coroutine lifecycle and cancellation

---

## Lesson 5: RememberUpdatedState - Preserving Latest Values

### What is RememberUpdatedState?

`RememberUpdatedState` is a specialized side effect that allows you to preserve the **latest value** of a variable or function in an effect, without restarting the effect when that value changes. This is particularly useful when you need to use a callback or value inside a long-running coroutine or effect.

### Basic Syntax

```kotlin
val updatedValue = rememberUpdatedState(newValue)

// Inside an effect
LaunchedEffect(Unit) {
    while (true) {
        // Use updatedValue.value to get the latest value
        processValue(updatedValue.value)
        delay(1000)
    }
}
```

### Key Characteristics

1. **Preserves latest values** - Updates without restarting the effect
2. **Prevents stale closures** - Ensures you always use the most recent value
3. **Works with any type** - Can be used with primitives, objects, functions, etc.
4. **Memory efficient** - Only stores references, not copies
5. **Composition-scoped** - Automatically cleaned up when composable leaves

### When to Use RememberUpdatedState

Use `RememberUpdatedState` when you need to:
- **Pass values to long-running effects** - Without restarting the effect
- **Use callbacks in effects** - While preserving the latest callback
- **Access changing values safely** - Without triggering recomposition loops
- **Create stable closures** - That can be passed to other components
- **Avoid unnecessary effect restarts** - When only some values change

### Simple Example: Latest Callback in Effect

Let's start with a simple example showing the problem and solution:

```kotlin
@Composable
fun ProblematicExample(callback: () -> Unit) {
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            callback() // This will always use the initial callback!
        }
    }
    
    Text("Timer running...")
}

@Composable
fun FixedExample(callback: () -> Unit) {
    val updatedCallback = rememberUpdatedState(callback)
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            updatedCallback.value() // This always uses the latest callback!
        }
    }
    
    Text("Timer running...")
}
```

**Problem Explanation:**
- In the problematic version, the `callback` parameter is captured in the closure when the effect first runs
- Even if the parent composable passes a different callback, the effect continues using the old one
- In the fixed version, `rememberUpdatedState` ensures the effect always uses the latest callback

### Example 1: Dynamic Countdown with Updated Configuration

```kotlin
@Composable
fun DynamicCountdown(
    initialTime: Int,
    onTick: (Int) -> Unit,
    onComplete: () -> Unit
) {
    var timeLeft by remember { mutableStateOf(initialTime) }
    val updatedOnTick = rememberUpdatedState(onTick)
    val updatedOnComplete = rememberUpdatedState(onComplete)
    
    LaunchedEffect(Unit) {
        while (timeLeft > 0) {
            delay(1000)
            timeLeft--
            updatedOnTick.value(timeLeft)
        }
        updatedOnComplete.value()
    }
    
    Column {
        Text("Time Left: $timeLeft")
        
        Row {
            Button(onClick = { timeLeft = 10 }) { Text("10s") }
            Button(onClick = { timeLeft = 30 }) { Text("30s") }
            Button(onClick = { timeLeft = 60 }) { Text("60s") }
        }
    }
}
```

**Usage Example:**
```kotlin
@Composable
fun CountdownDemo() {
    var message by remember { mutableStateOf("Ready") }
    
    DynamicCountdown(
        initialTime = 10,
        onTick = { remaining -> 
            message = "Countdown: $remaining"
        },
        onComplete = {
            message = "Time's up!"
        }
    )
    
    Text(message)
}
```

### Example 2: Progress Tracker with Dynamic Callbacks

```kotlin
@Composable
fun ProgressTracker(
    totalSteps: Int,
    onProgress: (Int, Int) -> Unit,
    onComplete: (Int) -> Unit
) {
    var currentStep by remember { mutableStateOf(0) }
    val updatedOnProgress = rememberUpdatedState(onProgress)
    val updatedOnComplete = rememberUpdatedState(onComplete)
    
    LaunchedEffect(Unit) {
        while (currentStep < totalSteps) {
            delay(1000)
            currentStep++
            updatedOnProgress.value(currentStep, totalSteps)
        }
        updatedOnComplete.value(totalSteps)
    }
    
    Column {
        Text("Step: $currentStep / $totalSteps")
        LinearProgressIndicator(
            progress = currentStep.toFloat() / totalSteps,
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
        )
        
        Row {
            Button(onClick = { currentStep = 0 }) { Text("Reset") }
            Button(onClick = { currentStep = minOf(currentStep + 5, totalSteps) }) { 
                Text("Skip 5") 
            }
        }
    }
}

// Usage
@Composable
fun ProgressDemo() {
    var progressMessage by remember { mutableStateOf("Starting...") }
    var completionData by remember { mutableStateOf<String?>(null) }
    
    ProgressTracker(
        totalSteps = 10,
        onProgress = { current, total ->
            progressMessage = "Progress: $current/$total (${((current.toFloat() / total) * 100).toInt()}%)"
        },
        onComplete = { finalStep ->
            completionData = "Completed at step $finalStep at ${System.currentTimeMillis()}"
        }
    )
    
    Text(progressMessage)
    completionData?.let { 
        Text(it, color = Color.Green) 
    }
}
```

### Example 3: Real-time Data Processor

```kotlin
data class SensorData(
    val temperature: Float,
    val humidity: Float,
    val timestamp: Long
)

@Composable
fun RealTimeDataProcessor(
    sensorData: Flow<SensorData>,
    onProcessed: (ProcessedData) -> Unit,
    processingInterval: Long = 1000L
) {
    val updatedOnProcessed = rememberUpdatedState(onProcessed)
    val scope = rememberCoroutineScope()
    
    LaunchedEffect(Unit) {
        sensorData.collect { data ->
            // Process the data
            val processed = processSensorData(data)
            updatedOnProcessed.value(ProcessedData(
                value = processed,
                timestamp = System.currentTimeMillis()
            ))
        }
    }
    
    // Also run periodic processing
    LaunchedEffect(Unit) {
        while (true) {
            delay(processingInterval)
            val currentTime = System.currentTimeMillis()
            // Could collect latest sensor data and process it
            // This is where updated callbacks ensure we use the latest functions
        }
    }
}

data class ProcessedData(
    val value: Float,
    val timestamp: Long
)

private fun processSensorData(data: SensorData): Float {
    return data.temperature * 0.8f + data.humidity * 0.2f
}
```

### Example 4: Dynamic Theme Updater

```kotlin
@Composable
fun DynamicThemeUpdater(
    theme: Theme,
    onThemeUpdate: (Theme) -> Unit
) {
    val updatedOnThemeUpdate = rememberUpdatedState(onThemeUpdate)
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(2000) // Update every 2 seconds
            
            // Simulate dynamic theme changes
            val currentTime = System.currentTimeMillis()
            val shouldUpdateTheme = (currentTime / 2000) % 2 == 0L
            
            if (shouldUpdateTheme) {
                val updatedTheme = theme.copy(
                    primaryColor = if (theme.primaryColor == Color.Blue) Color.Green else Color.Blue
                )
                updatedOnThemeUpdate.value(updatedTheme)
            }
        }
    }
    
    Text("Dynamic theme: ${theme.primaryColor}")
}

data class Theme(
    val primaryColor: Color,
    val secondaryColor: Color,
    val isDarkMode: Boolean
)
```

### Example 5: Network Request Manager with Dynamic Callbacks

```kotlin
data class NetworkRequest(
    val id: String,
    val url: String,
    val onSuccess: (String) -> Unit,
    val onError: (Exception) -> Unit
)

@Composable
fun NetworkRequestManager(
    requests: List<NetworkRequest>,
    maxConcurrentRequests: Int = 3
) {
    val activeRequests = remember { mutableStateMapOf<String, Job>() }
    val queue = remember { mutableStateListOf<NetworkRequest>() }
    val updatedQueue = rememberUpdatedState(queue)
    val scope = rememberCoroutineScope()
    
    // Add new requests to queue
    LaunchedEffect(requests) {
        requests.forEach { request ->
            if (!activeRequests.containsKey(request.id) && 
                !queue.any { it.id == request.id }) {
                queue.add(request)
            }
        }
    }
    
    // Process queue
    LaunchedEffect(Unit) {
        while (true) {
            if (activeRequests.size < maxConcurrentRequests && queue.isNotEmpty()) {
                val request = queue.removeAt(0)
                activeRequests[request.id] = processRequest(request)
            }
            delay(100) // Check queue every 100ms
        }
    }
    
    // Function to process a single request
    val processRequest: (NetworkRequest) -> Job = { request ->
        scope.launch {
            try {
                val result = withContext(Dispatchers.IO) {
                    // Use updated callbacks through rememberUpdatedState
                    simulateNetworkCall(request.url)
                }
                
                // Call the latest success callback
                request.onSuccess(result)
                
            } catch (e: Exception) {
                // Call the latest error callback
                request.onError(e)
            } finally {
                activeRequests.remove(request.id)
            }
        }
    }
    
    Text("Active: ${activeRequests.size}, Queued: ${queue.size}")
}

private suspend fun simulateNetworkCall(url: String): String {
    delay(Random.nextLong(1000, 5000)) // Simulate variable network delay
    if (Random.nextBoolean()) {
        return "Result from $url"
    } else {
        throw Exception("Network error for $url")
    }
}
```

### Example 6: Live Analytics Tracker

```kotlin
data class AnalyticsEvent(
    val eventName: String,
    val properties: Map<String, Any>,
    val timestamp: Long
)

@Composable
fun LiveAnalyticsTracker(
    isTracking: Boolean,
    onEventLogged: (AnalyticsEvent) -> Unit,
    onSessionEnd: (SessionData) -> Unit
) {
    val updatedOnEventLogged = rememberUpdatedState(onEventLogged)
    val updatedOnSessionEnd = rememberUpdatedState(onSessionEnd)
    val scope = rememberCoroutineScope()
    
    var sessionData by remember { 
        mutableStateOf(SessionData(startTime = System.currentTimeMillis()))
    }
    
    LaunchedEffect(isTracking) {
        if (isTracking) {
            // Start session
            sessionData = SessionData(startTime = System.currentTimeMillis())
            
            // Log session start
            updatedOnEventLogged.value(
                AnalyticsEvent(
                    eventName = "session_start",
                    properties = mapOf("session_id" to sessionData.sessionId),
                    timestamp = System.currentTimeMillis()
                )
            )
            
            // Periodic heartbeat
            while (isTracking) {
                delay(30000) // Send heartbeat every 30 seconds
                
                updatedOnEventLogged.value(
                    AnalyticsEvent(
                        eventName = "heartbeat",
                        properties = mapOf(
                            "session_id" to sessionData.sessionId,
                            "duration" to (System.currentTimeMillis() - sessionData.startTime)
                        ),
                        timestamp = System.currentTimeMillis()
                    )
                )
            }
            
            // Session ended
            val endTime = System.currentTimeMillis()
            val endData = sessionData.copy(endTime = endTime)
            
            updatedOnSessionEnd.value(endData)
        }
    }
    
    Text("Tracking: $isTracking")
    Text("Session: ${sessionData.sessionId}")
    Text("Duration: ${((System.currentTimeMillis() - sessionData.startTime) / 1000).toInt()}s")
}

data class SessionData(
    val sessionId: String = UUID.randomUUID().toString(),
    val startTime: Long,
    val endTime: Long? = null,
    val events: List<AnalyticsEvent> = emptyList()
)
```

### Common Patterns

#### Pattern 1: Callback Preservation Pattern

```kotlin
@Composable
fun CallbackPreserver<T, R>(
    input: T,
    processor: (T, (R) -> Unit) -> Unit,
    onResult: (R) -> Unit
) {
    val updatedOnResult = rememberUpdatedState(onResult)
    
    LaunchedEffect(input) {
        processor(input) { result ->
            updatedOnResult.value(result)
        }
    }
}
```

#### Pattern 2: Dynamic Configuration Pattern

```kotlin
@Composable
fun DynamicConfigProcessor(
    config: ProcessingConfig,
    onConfigApplied: (ProcessingResult) -> Unit
) {
    val updatedConfig = rememberUpdatedState(config)
    val updatedCallback = rememberUpdatedState(onConfigApplied)
    
    LaunchedEffect(Unit) {
        while (true) {
            val currentConfig = updatedConfig.value
            val result = processWithConfig(currentConfig)
            updatedCallback.value(result)
            delay(currentConfig.updateInterval)
        }
    }
}

data class ProcessingConfig(
    val mode: String,
    val sensitivity: Float,
    val updateInterval: Long
)
```

#### Pattern 3: State Synchronization Pattern

```kotlin
@Composable
fun StateSynchronizer(
    source: StateFlow<Any>,
    onSync: (Any) -> Unit
) {
    val updatedOnSync = rememberUpdatedState(onSync)
    
    LaunchedEffect(Unit) {
        source.collect { value ->
            updatedOnSync.value(value)
        }
    }
}
```

### Best Practices

1. **Use for callback preservation** - When callbacks might change in long-running effects
2. **Combine with other effects** - Works well with `LaunchedEffect` and `DisposableEffect`
3. **Avoid unnecessary updates** - Only use when you specifically need to preserve latest values
4. **Consider performance** - RememberUpdatedState has a small overhead
5. **Pair with proper cleanup** - Use with `DisposableEffect` when managing resources

### Common Mistakes

❌ **Wrong: Capturing values in closures**
```kotlin
@Composable
fun BadExample(onClick: () -> Unit) {
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            onClick() // Uses initial onClick forever
        }
    }
}
```

✅ **Correct: Using rememberUpdatedState**
```kotlin
@Composable
fun GoodExample(onClick: () -> Unit) {
    val updatedOnClick = rememberUpdatedState(onClick)
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            updatedOnClick.value() // Always uses latest onClick
        }
    }
}
```

❌ **Wrong: Unstable effect dependencies**
```kotlin
@Composable
fun BadExample(config: Config, processor: (Config) -> Result) {
    // This effect restarts every time config changes
    LaunchedEffect(config) {
        val result = processor(config)
        handleResult(result)
    }
}
```

✅ **Correct: Stable effect with updated state**
```kotlin
@Composable
fun GoodExample(config: Config, processor: (Config) -> Result) {
    val updatedConfig = rememberUpdatedState(config)
    val updatedProcessor = rememberUpdatedState(processor)
    
    LaunchedEffect(Unit) {
        while (true) {
            val currentConfig = updatedConfig.value
            val currentProcessor = updatedProcessor.value
            val result = currentProcessor(currentConfig)
            handleResult(result)
        }
    }
}
```

### Performance Considerations

1. **Minimal overhead** - RememberUpdatedState is very lightweight
2. **Memory efficient** - Only stores references, not copies
3. **Update on recomposition** - Value updates on composable recomposition
4. **Garbage collection** - Old values are eligible for GC when not referenced

### Advanced Example: Adaptive Learning System

```kotlin
data class LearningData(
    val userId: String,
    val contentId: String,
    val progress: Float,
    val timeSpent: Long
)

@Composable
fun AdaptiveLearningSystem(
    learningData: Flow<LearningData>,
    adaptToUser: (String, List<LearningData>) -> Adaptation,
    showRecommendation: (Adaptation) -> Unit,
    onLearningComplete: (String) -> Unit
) {
    val userProgress = remember { mutableStateMapOf<String, MutableList<LearningData>>() }
    val updatedShowRecommendation = rememberUpdatedState(showRecommendation)
    val updatedOnLearningComplete = rememberUpdatedState(onLearningComplete)
    val updatedAdaptToUser = rememberUpdatedState(adaptToUser)
    
    // Collect learning data
    LaunchedEffect(Unit) {
        learningData.collect { data ->
            val userId = data.userId
            val userData = userProgress.getOrPut(userId) { mutableListOf() }
            userData.add(data)
            
            // Get adaptation based on user's learning history
            val adaptation = updatedAdaptToUser.value(userId, userData.toList())
            updatedShowRecommendation.value(adaptation)
            
            // Check if learning is complete
            if (data.progress >= 1.0f) {
                updatedOnLearningComplete.value(data.contentId)
            }
        }
    }
    
    // Periodic adaptation check
    LaunchedEffect(Unit) {
        while (true) {
            delay(5000) // Check every 5 seconds
            
            userProgress.forEach { (userId, dataList) ->
                val adaptation = updatedAdaptToUser.value(userId, dataList.toList())
                updatedShowRecommendation.value(adaptation)
            }
        }
    }
}

data class Adaptation(
    val userId: String,
    val recommendedContent: List<String>,
    val difficulty: String,
    val nextSteps: List<String>
)
```

### Practice Exercise

Create a composable that:
1. Takes a list of users to monitor
2. For each user, starts a monitoring effect that tracks their activity
3. Allows the onUserUpdate callback to be updated dynamically
4. Uses rememberUpdatedState to ensure all effects use the latest callback
5. Properly cleans up when the composable leaves

**Hints:**
- Use `rememberUpdatedState` for the callback parameter
- Create a monitoring coroutine for each user
- Use `DisposableEffect` for cleanup
- Handle user additions/removals dynamically

### Key Takeaways

- `RememberUpdatedState` preserves the latest value without restarting effects
- Essential for callback preservation in long-running coroutines
- Prevents stale closures and ensures you always use current values
- Lightweight with minimal performance overhead
- Commonly used with `LaunchedEffect` for dynamic data processing
- Perfect for scenarios where values change but you don't want to restart the effect
- Works with any type: primitives, objects, functions, lambdas
- Enables stable closures that can be passed safely between components

---

## Lesson 6: SideEffect - Every Recomposition

### What is SideEffect?

`SideEffect` is a simple but powerful side effect that runs **on every successful recomposition** of a composable. Unlike other effects that manage coroutines or resources, `SideEffect` is primarily used for imperative operations that should happen whenever the UI updates, such as debugging, logging, analytics, and state synchronization.

### Basic Syntax

```kotlin
SideEffect {
    // Code to run on every recomposition
}
```

### Key Characteristics

1. **Runs on every recomposition** - Executes after successful composition
2. **No coroutine context** - Runs synchronously, not in a coroutine
3. **No cleanup** - No disposal or cancellation needed
4. **Lightweight** - Simple imperative operations only
5. **State-aware** - Has access to current state values

### When to Use SideEffect

Use `SideEffect` for:
- **Debug logging** - Track composable invocations and state
- **Analytics events** - Track UI interactions and state changes
- **State synchronization** - Sync UI state with external systems
- **Performance monitoring** - Track recomposition frequency
- **Memory leak detection** - Monitor composable lifecycle
- **External system updates** - Update non-UI systems based on state

### Simple Example: Debug Logging

Let's start with basic debugging to understand when composables recompose:

```kotlin
@Composable
fun DebugUserProfile(
    userId: String,
    userName: String
) {
    var isLoading by remember { mutableStateOf(false) }
    
    // Log composable invocations
    SideEffect {
        Log.d("UserProfile", "Composable recomposed - ID: $userId, Name: $userName, Loading: $isLoading")
    }
    
    Column {
        Text("User: $userName")
        Text("ID: $userId")
        Text("Loading: $isLoading")
        
        Button(onClick = { isLoading = !isLoading }) {
            Text("Toggle Loading")
        }
    }
}
```

**Explanation:**
- Every time the composable recomposes, the `SideEffect` runs
- This helps you understand when and why recompositions occur
- Useful for debugging performance issues

### Example 1: Analytics Event Tracking

```kotlin
@Composable
fun AnalyticsTrackingScreen(
    screenName: String,
    userId: String?,
    sessionStartTime: Long
) {
    var currentState by remember { mutableStateOf<ScreenState>(ScreenState.LOADING) }
    
    // Track screen views
    SideEffect {
        Analytics.trackEvent("screen_view", mapOf(
            "screen_name" to screenName,
            "user_id" to userId ?: "anonymous",
            "session_start" to sessionStartTime,
            "current_time" to System.currentTimeMillis(),
            "state" to currentState.name
        ))
    }
    
    // Track state changes
    SideEffect {
        Analytics.trackEvent("state_change", mapOf(
            "screen" to screenName,
            "new_state" to currentState.name,
            "timestamp" to System.currentTimeMillis()
        ))
    }
    
    Column {
        Text("Screen: $screenName")
        Text("State: ${currentState.name}")
        Text("Session: ${(System.currentTimeMillis() - sessionStartTime) / 1000}s")
        
        Button(onClick = { 
            currentState = when (currentState) {
                ScreenState.LOADING -> ScreenState.READY
                ScreenState.READY -> ScreenState.ACTIVE
                ScreenState.ACTIVE -> ScreenState.COMPLETED
                ScreenState.COMPLETED -> ScreenState.LOADING
            }
        }) {
            Text("Next State")
        }
    }
}

enum class ScreenState {
    LOADING, READY, ACTIVE, COMPLETED
}

// Mock Analytics class
object Analytics {
    fun trackEvent(eventName: String, properties: Map<String, Any>) {
        // In real app, this would send to your analytics service
        Log.d("Analytics", "$eventName: $properties")
    }
}
```

### Example 2: Performance Monitoring

```kotlin
@Composable
fun PerformanceMonitoredList(
    items: List<String>
) {
    var renderCount by remember { mutableStateOf(0) }
    var lastUpdateTime by remember { mutableStateOf(0L) }
    
    // Track render count
    SideEffect {
        renderCount++
        lastUpdateTime = System.currentTimeMillis()
        
        // Send performance metrics
        PerformanceMonitor.recordRender("ListComposable", renderCount)
    }
    
    // Monitor memory usage
    SideEffect {
        val runtime = Runtime.getRuntime()
        val usedMemory = runtime.totalMemory() - runtime.freeMemory()
        val maxMemory = runtime.maxMemory()
        
        if (usedMemory > maxMemory * 0.8) {
            Log.w("Performance", "High memory usage: ${(usedMemory * 100 / maxMemory)}%")
        }
    }
    
    Column {
        Text("Items: ${items.size}")
        Text("Renders: $renderCount")
        Text("Last Update: ${(System.currentTimeMillis() - lastUpdateTime)}ms ago")
        Text("Memory: ${items.size * 64} bytes")
        
        LazyColumn {
            items(items) { item ->
                Text(item, modifier = Modifier.padding(8.dp))
            }
        }
    }
}

// Mock Performance Monitor
object PerformanceMonitor {
    private val renderCounts = mutableMapOf<String, Int>()
    
    fun recordRender(composableName: String, count: Int) {
        renderCounts[composableName] = count
        Log.d("Performance", "$composableName: $count renders")
    }
    
    fun getRenderCount(composableName: String): Int {
        return renderCounts[composableName] ?: 0
    }
}
```

### Example 3: State Synchronization with External Systems

```kotlin
data class UserSettings(
    val theme: String,
    val notifications: Boolean,
    val language: String,
    val autoSave: Boolean
)

@Composable
fun UserSettingsSync(
    userId: String,
    settings: UserSettings
) {
    var lastSyncTime by remember { mutableStateOf(0L) }
    var isOnline by remember { mutableStateOf(true) }
    
    // Sync settings with backend
    SideEffect {
        if (isOnline) {
            // Update external storage
            ExternalSettingsManager.updateUserSettings(userId, settings)
            lastSyncTime = System.currentTimeMillis()
        }
    }
    
    // Track user interaction patterns
    SideEffect {
        val interaction = UserInteraction(
            type = "settings_view",
            timestamp = System.currentTimeMillis(),
            userId = userId,
            data = mapOf(
                "theme" to settings.theme,
                "notifications" to settings.notifications,
                "language" to settings.language
            )
        )
        
        InteractionTracker.recordInteraction(interaction)
    }
    
    // Monitor for conflicts
    SideEffect {
        val localSettings = LocalStorage.getUserSettings(userId)
        if (localSettings != null && localSettings != settings) {
            Log.w("Settings", "Settings conflict detected for user $userId")
            SettingsConflictResolver.handleConflict(userId, localSettings, settings)
        }
    }
    
    Column {
        Text("User: $userId")
        Text("Theme: ${settings.theme}")
        Text("Notifications: ${settings.notifications}")
        Text("Language: ${settings.language}")
        Text("Auto-save: ${settings.autoSave}")
        Text("Last Sync: ${(System.currentTimeMillis() - lastSyncTime) / 1000}s ago")
    }
}

// Mock external systems
object ExternalSettingsManager {
    fun updateUserSettings(userId: String, settings: UserSettings) {
        Log.d("ExternalSync", "Updated settings for $userId")
    }
}

object InteractionTracker {
    fun recordInteraction(interaction: UserInteraction) {
        Log.d("Interaction", "Recorded: ${interaction.type}")
    }
}

data class UserInteraction(
    val type: String,
    val timestamp: Long,
    val userId: String,
    val data: Map<String, Any>
)

object SettingsConflictResolver {
    fun handleConflict(userId: String, local: UserSettings, remote: UserSettings) {
        Log.d("Conflict", "Resolving settings conflict for $userId")
    }
}

object LocalStorage {
    fun getUserSettings(userId: String): UserSettings? {
        return null // Mock implementation
    }
}
```

### Example 4: Accessibility and Screen Reader Support

```kotlin
@Composable
fun AccessibleContent(
    title: String,
    content: String,
    imageUrl: String?
) {
    var currentAccessibilityMode by remember { mutableStateOf(AccessibilityMode.SCREEN_READER) }
    
    // Update accessibility announcements
    SideEffect {
        val announcement = when (currentAccessibilityMode) {
            AccessibilityMode.SCREEN_READER -> {
                "Content updated: $title. $content"
            }
            AccessibilityMode.HIGH_CONTRAST -> {
                "High contrast mode active. Title: $title"
            }
            AccessibilityMode.LARGE_TEXT -> {
                "Large text mode active"
            }
        }
        
        AccessibilityManager.announceForAccessibility(announcement)
    }
    
    // Update focus order for navigation
    SideEffect {
        val focusOrder = listOf(
            FocusableElement("title", title),
            FocusableElement("content", content),
            if (imageUrl != null) FocusableElement("image", "Image content") else null
        ).filterNotNull()
        
        AccessibilityManager.updateFocusOrder("content_screen", focusOrder)
    }
    
    Column {
        // Dynamic content based on accessibility mode
        when (currentAccessibilityMode) {
            AccessibilityMode.SCREEN_READER -> {
                Text(
                    text = title,
                    style = if (currentAccessibilityMode == AccessibilityMode.LARGE_TEXT) {
                        MaterialTheme.typography.h4
                    } else {
                        MaterialTheme.typography.h6
                    },
                    modifier = Modifier.semantics {
                        contentDescription = "Title: $title"
                    }
                )
                
                Text(
                    text = content,
                    style = if (currentAccessibilityMode == AccessibilityMode.LARGE_TEXT) {
                        MaterialTheme.typography.body1.copy(fontSize = 20.sp)
                    } else {
                        MaterialTheme.typography.body1
                    },
                    modifier = Modifier.semantics {
                        contentDescription = "Content: $content"
                    }
                )
            }
            AccessibilityMode.HIGH_CONTRAST -> {
                Text(
                    text = title,
                    color = Color.White,
                    style = MaterialTheme.typography.h6
                )
                Text(
                    text = content,
                    color = Color.White,
                    style = MaterialTheme.typography.body1
                )
            }
            AccessibilityMode.LARGE_TEXT -> {
                Text(
                    text = title,
                    style = MaterialTheme.typography.h4.copy(fontSize = 32.sp)
                )
                Text(
                    text = content,
                    style = MaterialTheme.typography.body1.copy(fontSize = 20.sp)
                )
            }
        }
        
        imageUrl?.let { url ->
            AsyncImage(
                model = url,
                contentDescription = "Main content image",
                modifier = Modifier
                    .size(200.dp)
                    .semantics {
                        contentDescription = "Content image"
                    }
            )
        }
        
        Row {
            AccessibilityMode.values().forEach { mode ->
                Button(
                    onClick = { currentAccessibilityMode = mode }
                ) {
                    Text(mode.name)
                }
            }
        }
    }
}

enum class AccessibilityMode {
    SCREEN_READER, HIGH_CONTRAST, LARGE_TEXT
}

// Mock accessibility manager
object AccessibilityManager {
    fun announceForAccessibility(message: String) {
        Log.d("Accessibility", "Announcement: $message")
    }
    
    fun updateFocusOrder(screenId: String, focusOrder: List<FocusableElement>) {
        Log.d("Accessibility", "Focus order updated for $screenId")
    }
}

data class FocusableElement(
    val id: String,
    val label: String
)
```

### Example 5: Theme and Style Synchronization

```kotlin
@Composable
fun ThemeSyncComponent(
    theme: AppTheme,
    isDarkMode: Boolean
) {
    var currentTheme by remember { mutableStateOf(theme) }
    var isDark by remember { mutableStateOf(isDarkMode) }
    
    // Sync theme with system
    SideEffect {
        val systemTheme = if (isSystemInDarkTheme()) "dark" else "light"
        val expectedTheme = if (isDark) "dark" else "light"
        
        if (systemTheme != expectedTheme) {
            Log.d("Theme", "System theme ($systemTheme) doesn't match user preference ($expectedTheme)")
        }
    }
    
    // Apply dynamic color changes
    SideEffect {
        val colorScheme = if (isDark) {
            darkColorScheme(primary = currentTheme.primaryColor)
        } else {
            lightColorScheme(primary = currentTheme.primaryColor)
        }
        
        // Update Material Theme if needed
        MaterialTheme.colorScheme = colorScheme
    }
    
    // Log theme changes for analytics
    SideEffect {
        Analytics.trackEvent("theme_applied", mapOf(
            "theme_name" to currentTheme.name,
            "is_dark" to isDark,
            "primary_color" to currentTheme.primaryColor.toString(),
            "timestamp" to System.currentTimeMillis()
        ))
    }
    
    Column {
        Text("Theme: ${currentTheme.name}")
        Text("Mode: ${if (isDark) "Dark" else "Light"}")
        Box(
            modifier = Modifier
                .size(50.dp)
                .background(currentTheme.primaryColor)
        )
        
        Row {
            Button(onClick = { currentTheme = AppTheme.DEFAULT }) { Text("Default") }
            Button(onClick = { currentTheme = AppTheme.CUSTOM }) { Text("Custom") }
            Button(onClick = { isDark = !isDark }) { 
                Text(if (isDark) "Light" else "Dark") 
            }
        }
    }
}

@Composable
private fun isSystemInDarkTheme(): Boolean {
    val context = LocalContext.current
    val uiMode = context.resources.configuration.uiMode and Configuration.UI_MODE_NIGHT_MASK
    return uiMode == Configuration.UI_MODE_NIGHT_YES
}

data class AppTheme(
    val name: String,
    val primaryColor: Color,
    val secondaryColor: Color
) {
    companion object {
        val DEFAULT = AppTheme("Default", Color.Blue, Color.LightGray)
        val CUSTOM = AppTheme("Custom", Color.Green, Color.Yellow)
    }
}
```

### Example 6: Error Boundary and Error Recovery

```kotlin
data class ErrorState(
    val hasError: Boolean,
    val errorMessage: String?,
    val errorCode: Int?,
    val recoverySuggestion: String?
)

@Composable
fun ErrorBoundaryWrapper(
    content: @Composable () -> Unit
) {
    var errorState by remember { mutableStateOf<ErrorState?>(null) }
    var errorCount by remember { mutableStateOf(0) }
    val errorLog = remember { mutableListOf<ErrorState>() }
    
    // Track error occurrences
    SideEffect {
        errorState?.let { error ->
            errorCount++
            errorLog.add(error)
            
            // Log error for debugging
            Log.e("ErrorBoundary", "Error #${errorCount}: ${error.errorMessage}", 
                Throwable("Error Code: ${error.errorCode}"))
            
            // Send to error reporting service
            ErrorReportingService.reportError(
                message = error.errorMessage,
                code = error.errorCode,
                stackTrace = Thread.currentThread().stackTrace
            )
        }
    }
    
    // Check for recurring errors
    SideEffect {
        if (errorLog.size > 10) {
            val recentErrors = errorLog.takeLast(10)
            val uniqueErrors = recentErrors.distinctBy { it.errorCode }
            
            if (uniqueErrors.size <= 2) {
                Log.w("ErrorBoundary", "Recurring errors detected: ${uniqueErrors.size} unique errors in last ${recentErrors.size} occurrences")
                
                // Implement recovery strategy
                implementRecoveryStrategy(uniqueErrors)
            }
        }
    }
    
    // Memory management for error logs
    SideEffect {
        if (errorLog.size > 100) {
            // Keep only recent errors to prevent memory issues
            errorLog.removeRange(0, errorLog.size - 50)
        }
    }
    
    Column {
        if (errorState != null) {
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                colors = CardDefaults.cardColors(
                    containerColor = Color.Red.copy(alpha = 0.1f)
                )
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text(
                        "Error Occurred",
                        style = MaterialTheme.typography.h6,
                        color = Color.Red
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    errorState?.errorMessage?.let { message ->
                        Text(message, color = Color.Red)
                    }
                    Spacer(modifier = Modifier.height(8.dp))
                    errorState?.recoverySuggestion?.let { suggestion ->
                        Text("Suggestion: $suggestion")
                    }
                    Spacer(modifier = Modifier.height(16.dp))
                    Row {
                        Button(onClick = {
                            errorState = null
                            errorLog.clear()
                        }) {
                            Text("Clear")
                        }
                        Button(onClick = {
                            errorState = null
                            implementRecoveryStrategy(listOf(errorState!!))
                        }) {
                            Text("Retry")
                        }
                    }
                }
            }
        } else {
            content()
        }
    }
    
    // Function to simulate error occurrence
    fun simulateError() {
        errorState = ErrorState(
            hasError = true,
            errorMessage = "Simulated error for testing",
            errorCode = 500,
            recoverySuggestion = "Check your network connection and try again"
        )
    }
}

// Mock error reporting service
object ErrorReportingService {
    fun reportError(
        message: String?,
        code: Int?,
        stackTrace: Array<StackTraceElement>
    ) {
        Log.d("ErrorReporting", "Error reported: $message (Code: $code)")
    }
}

private suspend fun implementRecoveryStrategy(errors: List<ErrorState>) {
    // Implement your recovery logic here
    Log.d("Recovery", "Implementing recovery strategy for ${errors.size} errors")
    delay(1000) // Simulate recovery process
}
```

### Common Patterns

#### Pattern 1: Debug Information Display

```kotlin
@Composable
fun DebugInfoDisplay(
    composableName: String,
    state: Any
) {
    var recompositionCount by remember { mutableStateOf(0) }
    
    SideEffect {
        recompositionCount++
        Log.d("Debug", "$composableName recomposed (${recompositionCount}x) with state: $state")
    }
    
    if (BuildConfig.DEBUG) {
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .padding(8.dp),
            backgroundColor = Color.Yellow.copy(alpha = 0.3f)
        ) {
            Column(modifier = Modifier.padding(8.dp)) {
                Text("DEBUG: $composableName")
                Text("Recompositions: $recompositionCount")
                Text("State: $state")
            }
        }
    }
}
```

#### Pattern 2: Performance Metrics Collection

```kotlin
@Composable
fun PerformanceMetricsCollector(
    componentName: String
) {
    var renderStartTime by remember { mutableStateOf(0L) }
    var averageRenderTime by remember { mutableStateOf(0.0) }
    var renderCount by remember { mutableStateOf(0) }
    
    SideEffect {
        renderStartTime = System.nanoTime()
    }
    
    DisposableEffect(Unit) {
        onDispose {
            val renderEndTime = System.nanoTime()
            val renderTime = (renderEndTime - renderStartTime) / 1_000_000.0 // ms
            
            renderCount++
            val totalTime = (averageRenderTime * (renderCount - 1)) + renderTime
            averageRenderTime = totalTime / renderCount
            
            Log.d("Performance", "$componentName: ${"%.2f".format(renderTime)}ms (avg: ${"%.2f".format(averageRenderTime)}ms)")
        }
    }
}
```

### Best Practices

1. **Use for non-async operations** - SideEffect runs synchronously
2. **Keep it lightweight** - Avoid heavy computations in SideEffect
3. **Use for debugging and monitoring** - Perfect for development and analytics
4. **Avoid state mutations** - Don't update state from SideEffect
5. **Consider performance** - Runs on every recomposition, so keep it simple
6. **Use conditional logging** - Only log in debug builds

### Common Mistakes

❌ **Wrong: Mutating state in SideEffect**
```kotlin
@Composable
fun BadExample() {
    var counter by remember { mutableStateOf(0) }
    
    SideEffect {
        counter++ // This could cause infinite recompositions!
    }
    
    Text("Counter: $counter")
}
```

✅ **Correct: Only read state in SideEffect**
```kotlin
@Composable
fun GoodExample() {
    var counter by remember { mutableStateOf(0) }
    
    SideEffect {
        Log.d("Counter", "Current value: $counter") // Read-only
    }
    
    Button(onClick = { counter++ }) {
        Text("Increment")
    }
}
```

❌ **Wrong: Heavy operations in SideEffect**
```kotlin
@Composable
fun BadExample(data: List<String>) {
    SideEffect {
        val result = heavyComputation(data) // Too heavy for every recomposition!
        processResult(result)
    }
    
    Text("Data: ${data.size}")
}
```

✅ **Correct: Use appropriate effect for heavy work**
```kotlin
@Composable
fun GoodExample(data: List<String>) {
    var result by remember { mutableStateOf<Any?>(null) }
    
    LaunchedEffect(data) {
        result = withContext(Dispatchers.Default) {
            heavyComputation(data)
        }
    }
    
    SideEffect {
        Log.d("Data", "Processing ${data.size} items")
    }
    
    Text("Data: ${data.size}")
}
```

### Performance Considerations

1. **Runs on every recomposition** - Keep it very lightweight
2. **Synchronous execution** - Can block UI if too heavy
3. **Memory allocation** - Avoid creating new objects
4. **I/O operations** - Use with caution, might impact performance
5. **Conditional execution** - Only run when needed

### Advanced Example: Comprehensive Monitoring System

```kotlin
@Composable
fun MonitoringSystem(
    screenName: String,
    userId: String?,
    features: List<String>
) {
    var sessionId by remember { mutableStateOf(UUID.randomUUID().toString()) }
    var screenStartTime by remember { mutableStateOf(System.currentTimeMillis()) }
    var lastActivityTime by remember { mutableStateOf(System.currentTimeMillis()) }
    var featureUsage by remember { mutableStateOf(mutableMapOf<String, Int>()) }
    
    // Session tracking
    SideEffect {
        val sessionDuration = System.currentTimeMillis() - screenStartTime
        if (sessionDuration > 300000) { // 5 minutes
            Log.d("Session", "Long session detected: ${sessionDuration / 1000}s")
        }
    }
    
    // Feature usage analytics
    SideEffect {
        features.forEach { feature ->
            val currentUsage = featureUsage[feature] ?: 0
            if (currentUsage > 0) {
                Analytics.trackEvent("feature_usage", mapOf(
                    "feature" to feature,
                    "usage_count" to currentUsage,
                    "screen" to screenName,
                    "user_id" to userId ?: "anonymous"
                ))
            }
        }
    }
    
    // Memory and performance monitoring
    SideEffect {
        val runtime = Runtime.getRuntime()
        val usedMemory = (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024 // MB
        val maxMemory = runtime.maxMemory() / 1024 / 1024 // MB
        
        if (usedMemory > maxMemory * 0.7) {
            Log.w("Memory", "High memory usage: ${usedMemory}MB / ${maxMemory}MB")
            
            // Trigger garbage collection if needed
            if (usedMemory > maxMemory * 0.8) {
                System.gc()
                Log.i("Memory", "Triggered garbage collection")
            }
        }
    }
    
    // Error and crash monitoring
    SideEffect {
        val uncaughtExceptions = Thread.getDefaultUncaughtExceptionHandler()
        if (uncaughtExceptions !is MonitoringExceptionHandler) {
            Thread.setDefaultUncaughtExceptionHandler(
                MonitoringExceptionHandler(screenName, userId)
            )
        }
    }
    
    // Accessibility monitoring
    SideEffect {
        val context = LocalContext.current
        val accessibilityManager = context.getSystemService(Context.ACCESSIBILITY_SERVICE) as AccessibilityManager
        
        if (accessibilityManager.isEnabled) {
            Analytics.trackEvent("accessibility_enabled", mapOf(
                "screen" to screenName,
                "user_id" to userId ?: "anonymous"
            ))
        }
    }
    
    // Network connectivity monitoring
    SideEffect {
        val context = LocalContext.current
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = connectivityManager.activeNetwork
        val capabilities = connectivityManager.getNetworkCapabilities(network)
        
        val networkType = when {
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true -> "WiFi"
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) == true -> "Cellular"
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET) == true -> "Ethernet"
            else -> "Unknown"
        }
        
        Log.d("Network", "Connected via: $networkType")
    }
    
    Column {
        Text("Screen: $screenName")
        Text("Session: $sessionId")
        Text("Duration: ${(System.currentTimeMillis() - screenStartTime) / 1000}s")
        Text("Features: ${features.joinToString()}")
        
        features.forEach { feature ->
            Button(
                onClick = {
                    featureUsage[feature] = (featureUsage[feature] ?: 0) + 1
                    lastActivityTime = System.currentTimeMillis()
                }
            ) {
                Text("Use $feature")
            }
        }
    }
}

// Exception handler for crash monitoring
class MonitoringExceptionHandler(
    private val screenName: String,
    private val userId: String?
) : Thread.UncaughtExceptionHandler {
    
    private val originalHandler = Thread.getDefaultUncaughtExceptionHandler()
    
    override fun uncaughtException(t: Thread, e: Throwable) {
        Log.e("Crash", "Uncaught exception in $screenName", e)
        
        CrashReportingService.reportCrash(
            thread = t.name,
            exception = e,
            screen = screenName,
            userId = userId
        )
        
        originalHandler?.uncaughtException(t, e)
    }
}

// Mock services
object Analytics {
    fun trackEvent(eventName: String, properties: Map<String, Any>) {
        Log.d("Analytics", "$eventName: $properties")
    }
}

object CrashReportingService {
    fun reportCrash(
        thread: String,
        exception: Throwable,
        screen: String,
        userId: String?
    ) {
        Log.d("CrashReport", "Crash in $screen by user $userId on thread $thread")
    }
}
```

### Practice Exercise

Create a composable that:
1. Tracks how many times it recomposes
2. Logs performance metrics (start time, end time, duration)
3. Monitors memory usage and logs warnings if high
4. Tracks user interactions and sends analytics events
5. Only runs in debug builds (use BuildConfig.DEBUG)
6. Provides a debug overlay showing all collected information

**Requirements:**
- Use `SideEffect` for logging and metrics
- Use `DisposableEffect` for cleanup and final reporting
- Use conditional compilation for debug-only features
- Create a comprehensive monitoring system

### Key Takeaways

- `SideEffect` runs on every successful recomposition
- Perfect for debugging, logging, and analytics
- Runs synchronously and cannot be cancelled
- Keep operations lightweight to avoid performance impact
- Excellent for state monitoring and external system updates
- Not suitable for heavy computations or async operations
- Combine with other effects for comprehensive monitoring
- Essential tool for performance optimization and debugging

---

## Lesson 7: Advanced Patterns - Complex Scenarios and Combinations

### Introduction to Advanced Patterns

In real-world applications, you'll often need to combine multiple side effects and handle complex scenarios that go beyond the basic use cases. This lesson covers architectural patterns, complex state management, and sophisticated combinations of side effects.

### Pattern 1: Effect Coordination and Dependency Management

Often, you need to coordinate multiple effects that depend on each other. Here's how to handle effect dependencies properly:

```kotlin
@Composable
fun CoordinatedEffects(
    userId: String,
    autoRefresh: Boolean,
    onDataLoaded: (UserData) -> Unit
) {
    var userData by remember { mutableStateOf<UserData?>(null) }
    var isLoading by remember { mutableStateOf(false) }
    var lastError by remember { mutableStateOf<String?>(null) }
    
    // Primary effect: Load user data
    LaunchedEffect(userId) {
        if (userId.isEmpty()) return@LaunchedEffect
        
        isLoading = true
        lastError = null
        
        try {
            userData = withContext(Dispatchers.IO) {
                fetchUserData(userId)
            }
            onDataLoaded(userData!!)
        } catch (e: Exception) {
            lastError = e.message
        } finally {
            isLoading = false
        }
    }
    
    // Secondary effect: Auto-refresh based on user data
    LaunchedEffect(userData, autoRefresh) {
        if (autoRefresh && userData != null) {
            while (autoRefresh && userData != null) {
                delay(30000) // Refresh every 30 seconds
                try {
                    val updatedData = withContext(Dispatchers.IO) {
                        fetchUserData(userId)
                    }
                    if (updatedData != userData) {
                        userData = updatedData
                        onDataLoaded(updatedData)
                    }
                } catch (e: Exception) {
                    Log.e("Refresh", "Failed to refresh data: ${e.message}")
                }
            }
        }
    }
    
    // Tertiary effect: Analytics and logging
    SideEffect {
        userData?.let { data ->
            Analytics.trackEvent("user_data_loaded", mapOf(
                "user_id" to userId,
                "data_type" to data.type,
                "auto_refresh" to autoRefresh,
                "timestamp" to System.currentTimeMillis()
            ))
        }
    }
    
    // Display logic
    when {
        isLoading -> LoadingIndicator()
        lastError != null -> ErrorDisplay(lastError!!)
        userData != null -> UserDataDisplay(userData!!)
    }
}
```

### Pattern 2: State Machine with Effects

Create a composable that manages complex state transitions with side effects:

```kotlin
sealed class AuthState {
    object Initial : AuthState()
    object Loading : AuthState()
    object Authenticated : AuthState()
    object Unauthenticated : AuthState()
    data class Error(val message: String) : AuthState()
}

@Composable
fun AuthStateMachine(
    initialState: AuthState = AuthState.Initial
) {
    var currentState by remember { mutableStateOf(initialState) }
    var sessionToken by remember { mutableStateOf<String?>(null) }
    var userProfile by remember { mutableStateOf<UserProfile?>(null) }
    val scope = rememberCoroutineScope()
    
    // Main authentication effect
    LaunchedEffect(currentState) {
        when (currentState) {
            is AuthState.Initial -> {
                // Check for existing session
                val existingToken = SessionManager.getStoredToken()
                if (existingToken != null) {
                    currentState = AuthState.Loading
                    validateSession(existingToken)
                } else {
                    currentState = AuthState.Unauthenticated
                }
            }
            is AuthState.Authenticated -> {
                // Load user profile
                sessionToken?.let { token ->
                    userProfile = withContext(Dispatchers.IO) {
                        fetchUserProfile(token)
                    }
                }
            }
            is AuthState.Error -> {
                // Log error for analytics
                SideEffect {
                    Analytics.trackEvent("auth_error", mapOf(
                        "error_message" to (currentState as AuthState.Error).message,
                        "previous_state" to currentState.toString()
                    ))
                }
            }
        }
    }
    
    // Session validation effect
    fun validateSession(token: String) {
        scope.launch {
            try {
                val isValid = withContext(Dispatchers.IO) {
                    AuthService.validateSession(token)
                }
                
                if (isValid) {
                    sessionToken = token
                    currentState = AuthState.Authenticated
                } else {
                    SessionManager.clearToken()
                    currentState = AuthState.Unauthenticated
                }
            } catch (e: Exception) {
                currentState = AuthState.Error("Session validation failed: ${e.message}")
            }
        }
    }
    
    // Session refresh effect
    LaunchedEffect(currentState) {
        if (currentState is AuthState.Authenticated) {
            while (currentState is AuthState.Authenticated) {
                delay(10 * 60 * 1000) // Refresh every 10 minutes
                
                sessionToken?.let { token ->
                    try {
                        val newToken = withContext(Dispatchers.IO) {
                            AuthService.refreshToken(token)
                        }
                        sessionToken = newToken
                        SessionManager.storeToken(newToken)
                    } catch (e: Exception) {
                        Log.e("Session", "Token refresh failed: ${e.message}")
                        currentState = AuthState.Unauthenticated
                    }
                }
            }
        }
    }
    
    // Cleanup effect
    DisposableEffect(Unit) {
        onDispose {
            // Clean up any ongoing authentication operations
            if (currentState is AuthState.Authenticated) {
                SideEffect {
                    Analytics.trackEvent("auth_cleanup", mapOf(
                        "cleanup_reason" to "composable_disposed",
                        "session_duration" to "unknown"
                    ))
                }
            }
        }
    }
    
    // UI based on state
    when (currentState) {
        is AuthState.Initial -> SplashScreen()
        is AuthState.Loading -> LoadingScreen()
        is AuthState.Authenticated -> {
            userProfile?.let { profile ->
                AuthenticatedScreen(
                    profile = profile,
                    onLogout = {
                        SessionManager.clearToken()
                        currentState = AuthState.Unauthenticated
                    }
                )
            } ?: LoadingScreen()
        }
        is AuthState.Unauthenticated -> LoginScreen(
            onLogin = { email, password ->
                currentState = AuthState.Loading
                scope.launch {
                    try {
                        val result = withContext(Dispatchers.IO) {
                            AuthService.login(email, password)
                        }
                        sessionToken = result.token
                        SessionManager.storeToken(result.token)
                        currentState = AuthState.Authenticated
                    } catch (e: Exception) {
                        currentState = AuthState.Error("Login failed: ${e.message}")
                    }
                }
            }
        )
        is AuthState.Error -> ErrorScreen(
            error = (currentState as AuthState.Error).message,
            onRetry = { currentState = AuthState.Initial }
        )
    }
}

// Mock services and data
data class UserProfile(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?
)

object SessionManager {
    fun getStoredToken(): String? = "mock_token"
    fun storeToken(token: String) { /* Store token */ }
    fun clearToken() { /* Clear token */ }
}

object AuthService {
    suspend fun validateSession(token: String): Boolean = true
    suspend fun refreshToken(token: String): String = "new_token"
    suspend fun login(email: String, password: String): AuthResult = AuthResult("token")
}

data class AuthResult(val token: String)
```

### Pattern 3: Complex Data Flow Management

Handle complex data flows with multiple sources and transformations:

```kotlin
data class DataSource<T>(
    val id: String,
    val data: Flow<T>,
    val cacheStrategy: CacheStrategy
)

enum class CacheStrategy {
    NO_CACHE, MEMORY_CACHE, DISK_CACHE, HYBRID_CACHE
}

@Composable
fun <T> MultiSourceDataManager(
    sources: List<DataSource<T>>,
    mergeStrategy: (List<T>) -> T,
    onDataMerged: (T) -> Unit,
    refreshInterval: Long = 30000L
) {
    var mergedData by remember { mutableStateOf<T?>(null) }
    var isLoading by remember { mutableStateOf(false) }
    var errorState by remember { mutableStateOf<Map<String, String?>>(emptyMap()) }
    var lastUpdateTime by remember { mutableStateOf(0L) }
    val scope = rememberCoroutineScope()
    
    // Data loading effect
    LaunchedEffect(sources) {
        isLoading = true
        errorState = emptyMap()
        
        val results = mutableListOf<T>()
        val errors = mutableMapOf<String, String?>()
        
        // Load from all sources concurrently
        coroutineScope {
            val jobs = sources.map { source ->
                async {
                    try {
                        val data = source.data.first()
                        results.add(data)
                    } catch (e: Exception) {
                        errors[source.id] = e.message
                    }
                }
            }
            
            jobs.forEach { job ->
                try {
                    job.await()
                } catch (e: Exception) {
                    errors["global"] = e.message
                }
            }
        }
        
        if (results.isNotEmpty()) {
            mergedData = mergeStrategy(results)
            lastUpdateTime = System.currentTimeMillis()
            onDataMerged(mergedData!!)
        }
        
        errorState = errors
        isLoading = false
    }
    
    // Auto-refresh effect
    LaunchedEffect(sources, refreshInterval) {
        while (true) {
            delay(refreshInterval)
            
            // Only refresh if not currently loading
            if (!isLoading) {
                val updatedSources = sources.map { source ->
                    source.copy(data = source.data)
                }
                
                // Trigger refresh by updating sources
                // This is a simplified example - in practice, you'd have more sophisticated logic
                Log.d("DataManager", "Auto-refreshing data from ${sources.size} sources")
            }
        }
    }
    
    // Cache management effect
    SideEffect {
        mergedData?.let { data ->
            sources.forEach { source ->
                when (source.cacheStrategy) {
                    CacheStrategy.MEMORY_CACHE -> {
                        MemoryCache.put(source.id, data)
                    }
                    CacheStrategy.DISK_CACHE -> {
                        scope.launch {
                            DiskCache.put(source.id, data)
                        }
                    }
                    CacheStrategy.HYBRID_CACHE -> {
                        MemoryCache.put(source.id, data)
                        scope.launch {
                            DiskCache.put(source.id, data)
                        }
                    }
                    CacheStrategy.NO_CACHE -> {
                        // No caching
                    }
                }
            }
        }
    }
    
    // Performance monitoring
    SideEffect {
        val now = System.currentTimeMillis()
        val timeSinceUpdate = now - lastUpdateTime
        
        if (timeSinceUpdate > refreshInterval * 1.5) {
            Log.w("DataManager", "Data is stale: ${timeSinceUpdate}ms since last update")
        }
        
        if (errorState.isNotEmpty()) {
            Log.e("DataManager", "Data source errors: $errorState")
        }
    }
    
    // Cleanup effect
    DisposableEffect(Unit) {
        onDispose {
            // Cancel any ongoing operations
            scope.coroutineContext.job.cancel()
            
            // Log analytics
            SideEffect {
                Analytics.trackEvent("data_manager_disposed", mapOf(
                    "sources_count" to sources.size,
                    "last_update_age" to (System.currentTimeMillis() - lastUpdateTime),
                    "error_count" to errorState.size
                ))
            }
        }
    }
    
    DataManagerDisplay(
        data = mergedData,
        isLoading = isLoading,
        errorState = errorState,
        sources = sources,
        lastUpdateTime = lastUpdateTime,
        onRefresh = {
            // Trigger manual refresh
            isLoading = true
        }
    )
}

// Mock cache implementations
object MemoryCache {
    private val cache = mutableMapOf<String, Any>()
    
    fun <T> put(key: String, data: T) {
        cache[key] = data
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <T> get(key: String): T? = cache[key] as? T
}

object DiskCache {
    suspend fun <T> put(key: String, data: T) {
        // Simulate disk I/O
        delay(100)
    }
}

@Composable
fun <T> DataManagerDisplay(
    data: T?,
    isLoading: Boolean,
    errorState: Map<String, String?>,
    sources: List<DataSource<T>>,
    lastUpdateTime: Long,
    onRefresh: () -> Unit
) {
    Column {
        Text("Data Manager")
        Text("Sources: ${sources.size}")
        Text("Last Update: ${SimpleDateFormat("HH:mm:ss").format(Date(lastUpdateTime))}")
        Text("Errors: ${errorState.size}")
        
        if (isLoading) {
            CircularProgressIndicator()
        } else if (data != null) {
            Text("Data: $data")
        } else {
            Text("No data available")
        }
        
        Button(onClick = onRefresh) {
            Text("Refresh")
        }
    }
}
```

### Pattern 4: Effect Composition for Complex Animations

Create sophisticated animation systems using multiple coordinated effects:

```kotlin
data class AnimationConfig(
    val duration: Long,
    val delay: Long = 0L,
    val easing: AnimationEasing = AnimationEasing.LINEAR,
    val repeatCount: Int = 1,
    val isReversible: Boolean = false
)

enum class AnimationEasing {
    LINEAR, EASE_IN, EASE_OUT, EASE_IN_OUT, BOUNCE
}

@Composable
fun ComplexAnimationSystem(
    animationConfig: AnimationConfig,
    isPlaying: Boolean,
    onAnimationComplete: (AnimationResult) -> Unit
) {
    var progress by remember { mutableStateOf(0f) }
    var currentPhase by remember { mutableStateOf(AnimationPhase.IDLE) }
    var animationStartTime by remember { mutableStateOf(0L) }
    var repeatCount by remember { mutableStateOf(0) }
    
    val updatedConfig = rememberUpdatedState(animationConfig)
    val updatedOnComplete = rememberUpdatedState(onAnimationComplete)
    val scope = rememberCoroutineScope()
    
    // Main animation loop
    LaunchedEffect(isPlaying, animationConfig) {
        if (!isPlaying) {
            currentPhase = AnimationPhase.IDLE
            return@LaunchedEffect
        }
        
        currentPhase = AnimationPhase.PLAYING
        animationStartTime = System.currentTimeMillis()
        repeatCount = 0
        
        while (isPlaying && currentPhase == AnimationPhase.PLAYING) {
            val phaseStartTime = System.currentTimeMillis()
            val adjustedDelay = updatedConfig.value.delay + (repeatCount * updatedConfig.value.duration / 2)
            
            // Handle initial delay
            if (adjustedDelay > 0) {
                delay(adjustedDelay)
            }
            
            // Animate forward
            animatePhase(
                from = 0f,
                to = 1f,
                duration = updatedConfig.value.duration,
                easing = updatedConfig.value.easing
            ) { phaseProgress ->
                progress = phaseProgress
                currentPhase = AnimationPhase.FORWARD
            }
            
            repeatCount++
            
            // Handle reverse animation if enabled
            if (updatedConfig.value.isReversible && repeatCount <= updatedConfig.value.repeatCount) {
                animatePhase(
                    from = 1f,
                    to = 0f,
                    duration = updatedConfig.value.duration,
                    easing = updatedConfig.value.easing
                ) { phaseProgress ->
                    progress = phaseProgress
                    currentPhase = AnimationPhase.REVERSE
                }
            }
            
            // Check completion
            if (repeatCount >= updatedConfig.value.repeatCount) {
                currentPhase = AnimationPhase.COMPLETED
                val result = AnimationResult(
                    totalDuration = System.currentTimeMillis() - animationStartTime,
                    repeatCount = repeatCount,
                    finalProgress = progress
                )
                updatedOnComplete.value(result)
                currentPhase = AnimationPhase.IDLE
                break
            }
        }
    }
    
    // Performance monitoring effect
    SideEffect {
        if (currentPhase == AnimationPhase.PLAYING) {
            val currentTime = System.currentTimeMillis()
            val elapsed = currentTime - animationStartTime
            val expectedProgress = (elapsed.toFloat() / (animationConfig.duration * animationConfig.repeatCount)).coerceIn(0f, 1f)
            val actualProgress = progress
            
            if (abs(actualProgress - expectedProgress) > 0.1f) {
                Log.w("Animation", "Progress drift detected: expected $expectedProgress, actual $actualProgress")
            }
        }
    }
    
    // Animation state display
    Column {
        Text("Animation System")
        Text("Phase: ${currentPhase.name}")
        Text("Progress: ${(progress * 100).toInt()}%")
        Text("Repeat: $repeatCount / ${animationConfig.repeatCount}")
        
        // Animated visual
        Box(
            modifier = Modifier
                .size(100.dp)
                .background(
                    color = Color.Blue.copy(alpha = 0.5f + progress * 0.5f),
                    shape = when {
                        progress < 0.3f -> CircleShape
                        progress < 0.7f -> RoundedCornerShape(8.dp)
                        else -> RectangleShape
                    }
                )
                .graphicsLayer(
                    rotationZ = progress * 360f,
                    scaleX = 0.5f + progress * 0.5f,
                    scaleY = 0.5f + progress * 0.5f
                )
        )
        
        Row {
            Button(onClick = { 
                scope.launch { 
                    // Manual control
                }
            }) {
                Text("Play")
            }
            Button(onClick = { 
                // Pause logic would go here
            }) {
                Text("Pause")
            }
            Button(onClick = { 
                // Reset logic would go here
            }) {
                Text("Reset")
            }
        }
    }
    
    // Animation phase helper
    suspend fun animatePhase(
        from: Float,
        to: Float,
        duration: Long,
        easing: AnimationEasing,
        onProgress: (Float) -> Unit
    ) {
        val startTime = System.currentTimeMillis()
        val endTime = startTime + duration
        
        while (System.currentTimeMillis() < endTime && currentPhase == AnimationPhase.PLAYING) {
            val currentTime = System.currentTimeMillis()
            val elapsed = currentTime - startTime
            val rawProgress = (elapsed.toFloat() / duration).coerceIn(0f, 1f)
            val easedProgress = applyEasing(rawProgress, easing)
            val currentValue = from + (to - from) * easedProgress
            
            onProgress(currentValue)
            delay(16) // ~60fps
        }
    }
    
    // Easing functions
    fun applyEasing(t: Float, easing: AnimationEasing): Float {
        return when (easing) {
            AnimationEasing.LINEAR -> t
            AnimationEasing.EASE_IN -> t * t * t
            AnimationEasing.EASE_OUT -> 1f - (1f - t).pow(3)
            AnimationEasing.EASE_IN_OUT -> if (t < 0.5f) {
                4f * t * t * t
            } else {
                1f - (-2f * t + 2f).pow(3) / 2f
            }
            AnimationEasing.BOUNCE -> when {
                t < 1f / 2.75f -> 7.5625f * t * t
                t < 2f / 2.75f -> {
                    val x = t - 1.5f / 2.75f
                    7.5625f * x * x + 0.75f
                }
                t < 2.5f / 2.75f -> {
                    val x = t - 2.25f / 2.75f
                    7.5625f * x * x + 0.9375f
                }
                else -> {
                    val x = t - 2.625f / 2.75f
                    7.5625f * x * x + 0.984375f
                }
            }
        }
    }
}

enum class AnimationPhase {
    IDLE, PLAYING, FORWARD, REVERSE, COMPLETED
}

data class AnimationResult(
    val totalDuration: Long,
    val repeatCount: Int,
    val finalProgress: Float
)
```

### Pattern 5: Effect-Based Dependency Injection

Create a composable that handles complex dependency injection with lifecycle awareness:

```kotlin
data class ServiceConfig(
    val baseUrl: String,
    val timeout: Long,
    val retryCount: Int,
    val cacheEnabled: Boolean
)

interface NetworkService {
    suspend fun getData(endpoint: String): String
    suspend fun postData(endpoint: String, data: Any): String
}

interface CacheService {
    suspend fun get(key: String): String?
    suspend fun put(key: String, value: String)
    suspend fun invalidate(key: String)
}

@Composable
fun <T> ServiceContainer(
    serviceConfig: ServiceConfig,
    serviceFactory: (ServiceConfig) -> T,
    usage: @Composable (T) -> Unit
) {
    var service by remember { mutableStateOf<T?>(null) }
    var isInitialized by remember { mutableStateOf(false) }
    var initError by remember { mutableStateOf<String?>(null) }
    
    // Service initialization effect
    LaunchedEffect(serviceConfig) {
        try {
            service = serviceFactory(serviceConfig)
            isInitialized = true
            initError = null
            
            Log.d("ServiceContainer", "Service initialized with config: $serviceConfig")
        } catch (e: Exception) {
            initError = e.message
            Log.e("ServiceContainer", "Failed to initialize service", e)
        }
    }
    
    // Health check effect
    LaunchedEffect(isInitialized) {
        if (isInitialized && service != null) {
            while (isInitialized && service != null) {
                try {
                    // Perform health check
                    val healthStatus = performHealthCheck(service!!)
                    if (!healthStatus.isHealthy) {
                        Log.w("ServiceContainer", "Service health check failed: ${healthStatus.message}")
                        // Could implement recovery logic here
                    }
                } catch (e: Exception) {
                    Log.e("ServiceContainer", "Health check error", e)
                }
                
                delay(30000) // Check every 30 seconds
            }
        }
    }
    
    // Configuration change effect
    SideEffect {
        if (isInitialized && service != null) {
            // Handle configuration changes
            Log.d("ServiceContainer", "Service is active with config: $serviceConfig")
        }
    }
    
    // Cleanup effect
    DisposableEffect(Unit) {
        onDispose {
            Log.d("ServiceContainer", "Cleaning up service")
            service = null
            isInitialized = false
        }
    }
    
    // Service usage
    when {
        !isInitialized && initError == null -> {
            Column(
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center,
                modifier = Modifier.fillMaxSize()
            ) {
                CircularProgressIndicator()
                Text("Initializing service...")
            }
        }
        initError != null -> {
            Column(
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center,
                modifier = Modifier.fillMaxSize()
            ) {
                Text("Service initialization failed", color = Color.Red)
                Text(initError!!, color = Color.Red)
                Button(onClick = {
                    // Retry initialization
                    initError = null
                }) {
                    Text("Retry")
                }
            }
        }
        service != null -> {
            usage(service!!)
        }
    }
}

data class HealthStatus(
    val isHealthy: Boolean,
    val message: String,
    val responseTime: Long
)

suspend fun <T> performHealthCheck(service: T): HealthStatus {
    val startTime = System.currentTimeMillis()
    return try {
        // Simulate health check
        delay(100)
        val responseTime = System.currentTimeMillis() - startTime
        HealthStatus(true, "Service is healthy", responseTime)
    } catch (e: Exception) {
        HealthStatus(false, "Health check failed: ${e.message}", System.currentTimeMillis() - startTime)
    }
}

// Usage example
@Composable
fun NetworkServiceExample() {
    val serviceConfig = ServiceConfig(
        baseUrl = "https://api.example.com",
        timeout = 5000L,
        retryCount = 3,
        cacheEnabled = true
    )
    
    ServiceContainer(
        serviceConfig = serviceConfig,
        serviceFactory = { config ->
            MockNetworkService(config.baseUrl)
        }
    ) { service: NetworkService ->
        ServiceUsageScreen(service)
    }
}

class MockNetworkService(private val baseUrl: String) : NetworkService {
    override suspend fun getData(endpoint: String): String {
        delay(200) // Simulate network call
        return "Data from $baseUrl/$endpoint"
    }
    
    override suspend fun postData(endpoint: String, data: Any): String {
        delay(300) // Simulate network call
        return "Posted to $baseUrl/$endpoint: $data"
    }
}

@Composable
fun ServiceUsageScreen(service: NetworkService) {
    var result by remember { mutableStateOf<String?>(null) }
    val scope = rememberCoroutineScope()
    
    Column {
        Text("Service Usage Example")
        
        Button(onClick = {
            scope.launch {
                try {
                    result = service.getData("users")
                } catch (e: Exception) {
                    result = "Error: ${e.message}"
                }
            }
        }) {
            Text("Get Data")
        }
        
        Button(onClick = {
            scope.launch {
                try {
                    result = service.postData("users", mapOf("name" to "John"))
                } catch (e: Exception) {
                    result = "Error: ${e.message}"
                }
            }
        }) {
            Text("Post Data")
        }
        
        result?.let {
            Text("Result: $it")
        }
    }
}
```

### Pattern 6: Effect-Based Event Bus

Create a composable-based event system with proper lifecycle management:

```kotlin
sealed class Event {
    data class UserAction(val action: String, val data: Any?) : Event()
    data class SystemEvent(val type: String, val payload: Map<String, Any>) : Event()
    data class ErrorEvent(val error: Throwable, val context: String) : Event()
    data class AnalyticsEvent(val eventName: String, val properties: Map<String, Any>) : Event()
}

typealias EventListener = (Event) -> Unit

class EventBus {
    private val listeners = mutableListOf<EventListener>()
    private val _events = MutableSharedFlow<Event>(extraBufferCapacity = 100)
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    fun subscribe(listener: EventListener) {
        listeners.add(listener)
    }
    
    fun unsubscribe(listener: EventListener) {
        listeners.remove(listener)
    }
    
    suspend fun publish(event: Event) {
        _events.emit(event)
        listeners.forEach { listener ->
            try {
                listener(event)
            } catch (e: Exception) {
                Log.e("EventBus", "Error in event listener", e)
            }
        }
    }
}

@Composable
fun EventBusProvider(
    eventBus: EventBus = remember { EventBus() },
    content: @Composable () -> Unit
) {
    var eventCount by remember { mutableStateOf(0) }
    var lastEvent by remember { mutableStateOf<Event?>(null) }
    var errorCount by remember { mutableStateOf(0) }
    
    // Event collection effect
    LaunchedEffect(Unit) {
        eventBus.events.collect { event ->
            eventCount++
            lastEvent = event
            
            when (event) {
                is Event.ErrorEvent -> {
                    errorCount++
                    Log.e("EventBus", "Error event in ${event.context}", event.error)
                }
                is Event.AnalyticsEvent -> {
                    // Send to analytics service
                    Analytics.trackEvent(event.eventName, event.properties)
                }
                else -> {
                    Log.d("EventBus", "Event received: $event")
                }
            }
        }
    }
    
    // Event bus management effect
    DisposableEffect(Unit) {
        Log.d("EventBus", "EventBusProvider created")
        
        onDispose {
            Log.d("EventBus", "EventBusProvider disposed. Total events: $eventCount, Errors: $errorCount")
        }
    }
    
    // Event bus state monitoring
    SideEffect {
        if (eventCount % 10 == 0 && eventCount > 0) {
            Log.d("EventBus", "Milestone: $eventCount events processed")
        }
    }
    
    CompositionLocalProvider(LocalEventBus provides eventBus) {
        content()
    }
}

@Composable
fun EventProducer(
    eventType: String,
    triggerOnMount: Boolean = false
) {
    val eventBus = LocalEventBus.current
    var eventCount by remember { mutableStateOf(0) }
    
    // Auto-trigger effect
    LaunchedEffect(triggerOnMount) {
        if (triggerOnMount) {
            val event = Event.SystemEvent(
                type = "auto_trigger",
                mapOf("event_type" to eventType, "timestamp" to System.currentTimeMillis())
            )
            eventBus.publish(event)
        }
    }
    
    Column {
        Text("Event Producer: $eventType")
        Text("Events sent: $eventCount")
        
        Row {
            Button(onClick = {
                val event = Event.UserAction(
                    action = "button_click",
                    data = mapOf("event_type" to eventType, "timestamp" to System.currentTimeMillis())
                )
                eventBus.publish(event)
                eventCount++
            }) {
                Text("Send User Action")
            }
            
            Button(onClick = {
                val event = Event.SystemEvent(
                    type = "manual_trigger",
                    mapOf("event_type" to eventType, "timestamp" to System.currentTimeMillis())
                )
                eventBus.publish(event)
                eventCount++
            }) {
                Text("Send System Event")
            }
        }
        
        Button(onClick = {
            try {
                throw Exception("Simulated error")
            } catch (e: Exception) {
                val event = Event.ErrorEvent(
                    error = e,
                    context = "EventProducer:$eventType"
                )
                eventBus.publish(event)
            }
        }) {
            Text("Simulate Error")
        }
    }
}

@Composable
fun EventConsumer() {
    val eventBus = LocalEventBus.current
    var lastEvents by remember { mutableStateOf<List<Event>>(emptyList()) }
    var isListening by remember { mutableStateOf(true) }
    
    // Event consumption effect
    val eventListener = rememberUpdatedState { event: Event ->
        if (isListening) {
            lastEvents = lastEvents.takeLast(9) + event // Keep last 10 events
        }
    }
    
    DisposableEffect(Unit) {
        eventBus.subscribe(eventListener.value)
        
        onDispose {
            eventBus.unsubscribe(eventListener.value)
        }
    }
    
    Column {
        Row(
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text("Event Consumer", style = MaterialTheme.typography.h6)
            Spacer(modifier = Modifier.width(8.dp))
            Switch(
                checked = isListening,
                onCheckedChange = { isListening = it }
            )
            Text(if (isListening) "Listening" else "Paused")
        }
        
        Text("Last ${lastEvents.size} events:")
        
        lastEvents.forEachIndexed { index, event ->
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(4.dp),
                backgroundColor = when (event) {
                    is Event.ErrorEvent -> Color.Red.copy(alpha = 0.1f)
                    is Event.AnalyticsEvent -> Color.Blue.copy(alpha = 0.1f)
                    is Event.UserAction -> Color.Green.copy(alpha = 0.1f)
                    is Event.SystemEvent -> Color.Orange.copy(alpha = 0.1f)
                }
            ) {
                Column(modifier = Modifier.padding(8.dp)) {
                    Text(
                        text = "${index + 1}. ${event::class.simpleName}",
                        style = MaterialTheme.typography.body2
                    )
                    Text(
                        text = when (event) {
                            is Event.UserAction -> "Action: ${event.action}"
                            is Event.SystemEvent -> "Type: ${event.type}"
                            is Event.ErrorEvent -> "Error: ${event.error.message}"
                            is Event.AnalyticsEvent -> "Analytics: ${event.eventName}"
                        },
                        style = MaterialTheme.typography.body2,
                        color = Color.Gray
                    )
                }
            }
        }
    }
}

// CompositionLocal for event bus
val LocalEventBus = compositionLocalOf<EventBus> {
    error("No EventBus provided")
}
```

### Best Practices for Advanced Patterns

1. **Effect Composition**: Always consider how effects interact with each other
2. **State Management**: Use proper state hoisting and data flow
3. **Performance**: Be mindful of the computational complexity of patterns
4. **Error Handling**: Implement comprehensive error boundaries and recovery
5. **Testing**: Write unit tests for complex effect combinations
6. **Documentation**: Document complex patterns and their dependencies
7. **Monitoring**: Add proper logging and analytics for production systems

### Common Anti-Patterns

❌ **Over-complication**: Using complex patterns for simple use cases
❌ **Tight Coupling**: Effects that depend too heavily on each other
❌ **Missing Error Handling**: Not accounting for failure scenarios
❌ **Resource Leaks**: Forgetting to clean up effects and resources
❌ **Performance Ignorance**: Not considering the computational cost of patterns

### Key Takeaways

- Advanced patterns combine multiple side effects for complex scenarios
- Effect coordination requires careful dependency management
- State machines provide robust state management for complex flows
- Data flow patterns handle multiple sources and transformations
- Animation systems benefit from coordinated effect management
- Dependency injection can be handled with lifecycle-aware effects
- Event bus patterns enable decoupled communication
- Always consider performance and error handling in complex patterns
- Document complex patterns thoroughly for maintenance
- Test effect combinations to ensure reliability

---

## Lesson 8: Real-world Applications - Complete Integration

### Introduction

In this final lesson, we'll build complete, production-ready applications that demonstrate how all the side effect concepts work together in real-world scenarios. These examples will show you how to architect complex apps using proper side effect management.

### Application 1: Task Management App with Real-time Sync

Let's build a complete task management application with real-time synchronization, offline support, and comprehensive state management:

```kotlin
// Main App Structure
@Composable
fun TaskManagementApp() {
    val context = LocalContext.current
    var currentUser by remember { mutableStateOf<User?>(null) }
    var isLoading by remember { mutableStateOf(true) }
    var syncStatus by remember { mutableStateOf(SyncStatus.IDLE) }
    val scope = rememberCoroutineScope()
    
    // Initialize app and user session
    LaunchedEffect(Unit) {
        try {
            currentUser = AuthManager.getCurrentUser()
            isLoading = false
        } catch (e: Exception) {
            Log.e("AppInit", "Failed to initialize app", e)
            isLoading = false
        }
    }
    
    // Real-time sync effect
    LaunchedEffect(currentUser) {
        currentUser?.let { user ->
            startRealTimeSync(user.id)
        }
    }
    
    // Performance monitoring
    SideEffect {
        if (BuildConfig.DEBUG) {
            Log.d("App", "Current user: ${currentUser?.id ?: "Anonymous"}, Sync: $syncStatus")
        }
    }
    
    when {
        isLoading -> AppLoadingScreen()
        currentUser == null -> AuthScreen(
            onLogin = { email, password ->
                scope.launch {
                    try {
                        isLoading = true
                        val user = AuthManager.login(email, password)
                        currentUser = user
                    } catch (e: Exception) {
                        // Handle login error
                    } finally {
                        isLoading = false
                    }
                }
            }
        )
        else -> TaskManagerRoot(
            user = currentUser!!,
            syncStatus = syncStatus,
            onSyncStatusChange = { syncStatus = it }
        )
    }
}

@Composable
fun TaskManagerRoot(
    user: User,
    syncStatus: SyncStatus,
    onSyncStatusChange: (SyncStatus) -> Unit
) {
    var tasks by remember { mutableStateOf<List<Task>>(emptyList()) }
    var selectedTask by remember { mutableStateOf<Task?>(null) }
    var isOnline by remember { mutableStateOf(true) }
    val scope = rememberCoroutineScope()
    
    // Network connectivity monitoring
    DisposableEffect(Unit) {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                isOnline = true
                onSyncStatusChange(SyncStatus.ONLINE)
            }
            
            override fun onLost(network: Network) {
                isOnline = false
                onSyncStatusChange(SyncStatus.OFFLINE)
            }
        }
        
        connectivityManager.registerDefaultNetworkCallback(networkCallback)
        
        onDispose {
            connectivityManager.unregisterNetworkCallback(networkCallback)
        }
    }
    
    // Load tasks effect
    LaunchedEffect(user.id) {
        tasks = withContext(Dispatchers.IO) {
            TaskRepository.getTasks(user.id)
        }
    }
    
    // Real-time sync effect
    LaunchedEffect(user.id, isOnline) {
        if (isOnline) {
            startTaskSync(user.id, onSyncStatusChange)
        }
    }
    
    // Task modification effect
    val updatedOnTaskChange = rememberUpdatedState { task: Task, action: TaskAction ->
        scope.launch {
            when (action) {
                is TaskAction.Create -> {
                    val newTask = withContext(Dispatchers.IO) {
                        TaskRepository.createTask(user.id, action.taskData)
                    }
                    tasks = tasks + newTask
                }
                is TaskAction.Update -> {
                    withContext(Dispatchers.IO) {
                        TaskRepository.updateTask(user.id, action.taskId, action.updates)
                    }
                    tasks = tasks.map { 
                        if (it.id == action.taskId) it.copy(
                            title = action.updates.title ?: it.title,
                            description = action.updates.description ?: it.description,
                            dueDate = action.updates.dueDate ?: it.dueDate,
                            priority = action.updates.priority ?: it.priority,
                            isCompleted = action.updates.isCompleted ?: it.isCompleted,
                            lastModified = System.currentTimeMillis()
                        ) else it
                    }
                }
                is TaskAction.Delete -> {
                    withContext(Dispatchers.IO) {
                        TaskRepository.deleteTask(user.id, action.taskId)
                    }
                    tasks = tasks.filter { it.id != action.taskId }
                }
            }
        }
    }
    
    // Task list display
    TaskListScreen(
        tasks = tasks,
        user = user,
        isOnline = isOnline,
        syncStatus = syncStatus,
        selectedTask = selectedTask,
        onTaskSelected = { selectedTask = it },
        onTaskAction = updatedOnTaskChange.value,
        onAddTask = {
            // Navigate to task creation
        },
        onSync = {
            scope.launch {
                onSyncStatusChange(SyncStatus.SYNCING)
                try {
                    withContext(Dispatchers.IO) {
                        TaskRepository.syncTasks(user.id)
                    }
                    onSyncStatusChange(SyncStatus.ONLINE)
                } catch (e: Exception) {
                    onSyncStatusChange(SyncStatus.ERROR)
                }
            }
        }
    )
}

@Composable
fun TaskListScreen(
    tasks: List<Task>,
    user: User,
    isOnline: Boolean,
    syncStatus: SyncStatus,
    selectedTask: Task?,
    onTaskSelected: (Task) -> Unit,
    onTaskAction: (Task, TaskAction) -> Unit,
    onAddTask: () -> Unit,
    onSync: () -> Unit
) {
    var searchQuery by remember { mutableStateOf("") }
    var filter by remember { mutableStateOf(TaskFilter.ALL) }
    var sortBy by remember { mutableStateOf(TaskSort.DUE_DATE) }
    
    // Search and filter effect
    val filteredTasks = remember(tasks, searchQuery, filter, sortBy) {
        var result = tasks
        
        // Apply search
        if (searchQuery.isNotEmpty()) {
            result = result.filter { 
                it.title.contains(searchQuery, ignoreCase = true) ||
                it.description?.contains(searchQuery, ignoreCase = true) == true
            }
        }
        
        // Apply filter
        result = when (filter) {
            TaskFilter.ALL -> result
            TaskFilter.ACTIVE -> result.filter { !it.isCompleted }
            TaskFilter.COMPLETED -> result.filter { it.isCompleted }
            TaskFilter.HIGH_PRIORITY -> result.filter { it.priority == TaskPriority.HIGH }
        }
        
        // Apply sort
        result = when (sortBy) {
            TaskSort.DUE_DATE -> result.sortedBy { it.dueDate }
            TaskSort.PRIORITY -> result.sortedByDescending { it.priority.ordinal }
            TaskSort.CREATED -> result.sortedByDescending { it.createdAt }
            TaskSort.TITLE -> result.sortedBy { it.title }
        }
        
        result
    }
    
    // Analytics tracking
    SideEffect {
        Analytics.trackEvent("task_list_viewed", mapOf(
            "user_id" to user.id,
            "task_count" to tasks.size,
            "is_online" to isOnline,
            "sync_status" to syncStatus.name
        ))
    }
    
    Column {
        // Header with sync status
        TaskListHeader(
            user = user,
            isOnline = isOnline,
            syncStatus = syncStatus,
            onSync = onSync
        )
        
        // Search and filters
        TaskListControls(
            searchQuery = searchQuery,
            onSearchChange = { searchQuery = it },
            filter = filter,
            onFilterChange = { filter = it },
            sortBy = sortBy,
            onSortChange = { sortBy = it }
        )
        
        // Task list
        LazyColumn {
            items(filteredTasks) { task ->
                TaskItem(
                    task = task,
                    isSelected = selectedTask?.id == task.id,
                    onClick = { onTaskSelected(task) },
                    onToggleComplete = {
                        onTaskAction(task, TaskAction.Update(
                            taskId = task.id,
                            updates = TaskUpdate(isCompleted = !task.isCompleted)
                        ))
                    },
                    onDelete = {
                        onTaskAction(task, TaskAction.Delete(task.id))
                    }
                )
            }
        }
        
        // Add task button
        FloatingActionButton(
            onClick = onAddTask,
            modifier = Modifier
                .align(Alignment.End)
                .padding(16.dp)
        ) {
            Icon(Icons.Default.Add, contentDescription = "Add Task")
        }
    }
}

@Composable
fun TaskListHeader(
    user: User,
    isOnline: Boolean,
    syncStatus: SyncStatus,
    onSync: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Column {
            Text(
                text = "Welcome, ${user.name}",
                style = MaterialTheme.typography.h6
            )
            Text(
                text = "${user.taskCount} tasks remaining",
                style = MaterialTheme.typography.body2,
                color = Color.Gray
            )
        }
        
        Row(verticalAlignment = Alignment.CenterVertically) {
            // Sync status indicator
            Icon(
                imageVector = when {
                    syncStatus == SyncStatus.SYNCING -> Icons.Default.Sync
                    isOnline -> Icons.Default.Wifi
                    else -> Icons.Default.WifiOff
                },
                contentDescription = "Sync Status",
                tint = when {
                    syncStatus == SyncStatus.ERROR -> Color.Red
                    !isOnline -> Color.Gray
                    else -> Color.Green
                }
            )
            
            Spacer(modifier = Modifier.width(8.dp))
            
            // Sync button
            IconButton(onClick = onSync) {
                Icon(
                    imageVector = Icons.Default.Refresh,
                    contentDescription = "Sync"
                )
            }
        }
    }
}

@Composable
fun TaskItem(
    task: Task,
    isSelected: Boolean,
    onClick: () -> Unit,
    onToggleComplete: () -> Unit,
    onDelete: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp)
            .clickable { onClick() }
            .border(
                width = if (isSelected) 2.dp else 0.dp,
                color = if (isSelected) MaterialTheme.colors.primary else Color.Transparent,
                shape = RoundedCornerShape(8.dp)
            ),
        elevation = if (isSelected) 8.dp else 4.dp
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Column(modifier = Modifier.weight(1f)) {
                    Text(
                        text = task.title,
                        style = MaterialTheme.typography.subtitle1,
                        textDecoration = if (task.isCompleted) TextDecoration.LineThrough else null
                    )
                    
                    if (!task.description.isNullOrEmpty()) {
                        Text(
                            text = task.description!!,
                            style = MaterialTheme.typography.body2,
                            color = Color.Gray,
                            maxLines = 2,
                            overflow = TextOverflow.Ellipsis
                        )
                    }
                    
                    Row(
                        verticalAlignment = Alignment.CenterVertically,
                        modifier = Modifier.padding(top = 8.dp)
                    ) {
                        // Priority indicator
                        Box(
                            modifier = Modifier
                                .size(12.dp)
                                .background(
                                    color = when (task.priority) {
                                        TaskPriority.HIGH -> Color.Red
                                        TaskPriority.MEDIUM -> Color.Orange
                                        TaskPriority.LOW -> Color.Green
                                    },
                                    shape = CircleShape
                                )
                        )
                        
                        Spacer(modifier = Modifier.width(8.dp))
                        
                        Text(
                            text = task.priority.name,
                            style = MaterialTheme.typography.caption
                        )
                        
                        task.dueDate?.let { dueDate ->
                            Spacer(modifier = Modifier.width(16.dp))
                            Text(
                                text = "Due: ${formatDate(dueDate)}",
                                style = MaterialTheme.typography.caption
                            )
                        }
                    }
                }
                
                Column {
                    // Complete checkbox
                    Checkbox(
                        checked = task.isCompleted,
                        onCheckedChange = { onToggleComplete() }
                    )
                    
                    // Delete button
                    IconButton(onClick = onDelete) {
                        Icon(
                            imageVector = Icons.Default.Delete,
                            contentDescription = "Delete",
                            tint = Color.Red
                        )
                    }
                }
            }
        }
    }
}

// Supporting data classes and enums
data class User(
    val id: String,
    val name: String,
    val email: String,
    val taskCount: Int
)

data class Task(
    val id: String,
    val title: String,
    val description: String?,
    val dueDate: Long?,
    val priority: TaskPriority,
    val isCompleted: Boolean,
    val createdAt: Long,
    val lastModified: Long,
    val userId: String
)

enum class TaskPriority { HIGH, MEDIUM, LOW }
enum class TaskFilter { ALL, ACTIVE, COMPLETED, HIGH_PRIORITY }
enum class TaskSort { DUE_DATE, PRIORITY, CREATED, TITLE }
enum class SyncStatus { IDLE, SYNCING, ONLINE, OFFLINE, ERROR }

sealed class TaskAction {
    data class Create(val taskData: CreateTaskData) : TaskAction()
    data class Update(val taskId: String, val updates: TaskUpdate) : TaskAction()
    data class Delete(val taskId: String) : TaskAction()
}

data class CreateTaskData(
    val title: String,
    val description: String? = null,
    val dueDate: Long? = null,
    val priority: TaskPriority = TaskPriority.MEDIUM
)

data class TaskUpdate(
    val title: String? = null,
    val description: String? = null,
    val dueDate: Long? = null,
    val priority: TaskPriority? = null,
    val isCompleted: Boolean? = null
)

// Repository and services
object TaskRepository {
    suspend fun getTasks(userId: String): List<Task> {
        // Simulate database call
        delay(200)
        return emptyList() // Return actual tasks
    }
    
    suspend fun createTask(userId: String, taskData: CreateTaskData): Task {
        delay(100)
        return Task(
            id = UUID.randomUUID().toString(),
            title = taskData.title,
            description = taskData.description,
            dueDate = taskData.dueDate,
            priority = taskData.priority,
            isCompleted = false,
            createdAt = System.currentTimeMillis(),
            lastModified = System.currentTimeMillis(),
            userId = userId
        )
    }
    
    suspend fun updateTask(userId: String, taskId: String, updates: TaskUpdate) {
        delay(100)
        // Update task in database
    }
    
    suspend fun deleteTask(userId: String, taskId: String) {
        delay(100)
        // Delete task from database
    }
    
    suspend fun syncTasks(userId: String) {
        // Sync with remote server
        delay(500)
    }
}

object AuthManager {
    suspend fun getCurrentUser(): User? {
        delay(200)
        return User("1", "John Doe", "john@example.com", 5)
    }
    
    suspend fun login(email: String, password: String): User {
        delay(500)
        return User("1", "John Doe", email, 5)
    }
}

suspend fun startTaskSync(userId: String, onSyncStatusChange: (SyncStatus) -> Unit) {
    // Implement real-time sync
    onSyncStatusChange(SyncStatus.ONLINE)
}

// Mock Analytics
object Analytics {
    fun trackEvent(eventName: String, properties: Map<String, Any>) {
        Log.d("Analytics", "$eventName: $properties")
    }
}

// Date formatting
@Composable
fun formatDate(timestamp: Long): String {
    return remember(timestamp) {
        SimpleDateFormat("MMM dd, yyyy", Locale.getDefault()).format(Date(timestamp))
    }
}
```

### Application 2: Real-time Chat Application

Now let's build a complete real-time chat application with message handling, typing indicators, and media sharing:

```kotlin
@Composable
fun ChatApplication() {
    var currentUser by remember { mutableStateOf<User?>(null) }
    var currentChat by remember { mutableStateOf<ChatRoom?>(null) }
    var isOnline by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()
    
    // App initialization
    LaunchedEffect(Unit) {
        currentUser = UserManager.getCurrentUser()
        isOnline = NetworkManager.isConnected()
    }
    
    // Online status monitoring
    DisposableEffect(Unit) {
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                isOnline = true
                scope.launch { reconnectToChat() }
            }
            
            override fun onLost(network: Network) {
                isOnline = false
            }
        }
        
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        connectivityManager.registerDefaultNetworkCallback(networkCallback)
        
        onDispose {
            connectivityManager.unregisterNetworkCallback(networkCallback)
        }
    }
    
    when {
        currentUser == null -> ChatAuthScreen()
        currentChat == null -> ChatListScreen(
            currentUser = currentUser!!,
            onChatSelected = { currentChat = it }
        )
        else -> ChatRoomScreen(
            currentUser = currentUser!!,
            chatRoom = currentChat!!,
            isOnline = isOnline,
            onBack = { currentChat = null }
        )
    }
}

@Composable
fun ChatRoomScreen(
    currentUser: User,
    chatRoom: ChatRoom,
    isOnline: Boolean,
    onBack: () -> Unit
) {
    var messages by remember { mutableStateOf<List<Message>>(emptyList()) }
    var newMessage by remember { mutableStateOf("") }
    var isTyping by remember { mutableStateOf(false) }
    var typingUsers by remember { mutableStateOf<Set<String>>(emptySet()) }
    var isRecording by remember { mutableStateOf(false) }
    var recordingTime by remember { mutableStateOf(0) }
    val scope = rememberCoroutineScope()
    
    val updatedChatRoom = rememberUpdatedState(chatRoom)
    val updatedCurrentUser = rememberUpdatedState(currentUser)
    
    // Load messages effect
    LaunchedEffect(chatRoom.id) {
        messages = withContext(Dispatchers.IO) {
            ChatRepository.getMessages(chatRoom.id)
        }
    }
    
    // Real-time message updates
    LaunchedEffect(chatRoom.id) {
        ChatRepository.subscribeToMessages(chatRoom.id) { message ->
            messages = messages + message
        }
    }
    
    // Typing indicators
    LaunchedEffect(chatRoom.id) {
        ChatRepository.subscribeToTyping(chatRoom.id) { userId, isTypingUser ->
            typingUsers = if (isTypingUser) {
                typingUsers + userId
            } else {
                typingUsers - userId
            }
        }
    }
    
    // Auto-typing detection
    val debounceTimeout = remember { mutableStateOf(2000L) }
    val typingJob = remember { mutableStateOf<Job?>(null) }
    
    DisposableEffect(newMessage) {
        if (newMessage.isNotEmpty()) {
            // Send typing started
            ChatRepository.sendTyping(chatRoom.id, currentUser.id, true)
            
            // Cancel previous debounce job
            typingJob.value?.cancel()
            
            // Start new debounce job
            typingJob.value = scope.launch {
                delay(debounceTimeout.value)
                ChatRepository.sendTyping(chatRoom.id, currentUser.id, false)
                typingUsers = typingUsers - currentUser.id
            }
        } else {
            // Cancel typing
            typingJob.value?.cancel()
            ChatRepository.sendTyping(chatRoom.id, currentUser.id, false)
        }
        
        onDispose {
            typingJob.value?.cancel()
        }
    }
    
    // Voice recording effect
    LaunchedEffect(isRecording) {
        if (isRecording) {
            while (isRecording) {
                delay(1000)
                recordingTime++
            }
        } else {
            recordingTime = 0
        }
    }
    
    // Message send effect
    val updatedOnSendMessage = rememberUpdatedState { messageText: String, messageType: MessageType ->
        scope.launch {
            if (messageText.isNotEmpty()) {
                val message = Message(
                    id = UUID.randomUUID().toString(),
                    content = messageText,
                    type = messageType,
                    senderId = currentUser.id,
                    senderName = currentUser.name,
                    timestamp = System.currentTimeMillis(),
                    chatRoomId = chatRoom.id,
                    isRead = false
                )
                
                withContext(Dispatchers.IO) {
                    ChatRepository.sendMessage(message)
                }
                
                newMessage = ""
            }
        }
    }
    
    // Cleanup effect
    DisposableEffect(Unit) {
        onDispose {
            ChatRepository.unsubscribeFromMessages(chatRoom.id)
            ChatRepository.unsubscribeFromTyping(chatRoom.id)
        }
    }
    
    Column(modifier = Modifier.fillMaxSize()) {
        // Chat header
        ChatRoomHeader(
            chatRoom = chatRoom,
            isOnline = isOnline,
            typingUsers = typingUsers,
            onBack = onBack
        )
        
        // Messages list
        LazyColumn(
            modifier = Modifier
                .weight(1f)
                .fillMaxWidth()
        ) {
            items(messages) { message ->
                MessageBubble(
                    message = message,
                    isCurrentUser = message.senderId == currentUser.id,
                    onLongPress = {
                        // Show message options
                    }
                )
            }
        }
        
        // Typing indicator
        if (typingUsers.isNotEmpty()) {
            TypingIndicator(
                userNames = typingUsers.mapNotNull { userId ->
                    chatRoom.participants.find { it.id == userId }?.name
                },
                modifier = Modifier.padding(horizontal = 16.dp, vertical = 4.dp)
            )
        }
        
        // Message input
        MessageInput(
            messageText = newMessage,
            onMessageChange = { newMessage = it },
            onSendMessage = updatedOnSendMessage.value,
            isRecording = isRecording,
            onStartRecording = { isRecording = true },
            onStopRecording = { 
                isRecording = false
                // Handle voice message
            },
            isOnline = isOnline
        )
    }
}

@Composable
fun MessageBubble(
    message: Message,
    isCurrentUser: Boolean,
    onLongPress: () -> Unit
) {
    val isImage = message.type == MessageType.IMAGE
    val isVoice = message.type == MessageType.VOICE
    
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp),
        horizontalArrangement = if (isCurrentUser) Arrangement.End else Arrangement.Start
    ) {
        Column(
            modifier = Modifier
                .widthIn(max = 280.dp)
                .clickable { /* Handle click */ }
                .longPressModifier { onLongPress() }
                .background(
                    color = if (isCurrentUser) {
                        MaterialTheme.colors.primary
                    } else {
                        Color.LightGray
                    },
                    shape = RoundedCornerShape(
                        topStart = if (isCurrentUser) 16.dp else 4.dp,
                        topEnd = 16.dp,
                        bottomStart = 4.dp,
                        bottomEnd = if (isCurrentUser) 4.dp else 16.dp
                    )
                )
                .padding(12.dp)
        ) {
            if (!isCurrentUser) {
                Text(
                    text = message.senderName,
                    style = MaterialTheme.typography.caption,
                    color = Color.Gray
                )
            }
            
            when (message.type) {
                MessageType.TEXT -> {
                    Text(
                        text = message.content,
                        style = MaterialTheme.typography.body2,
                        color = if (isCurrentUser) Color.White else Color.Black
                    )
                }
                MessageType.IMAGE -> {
                    AsyncImage(
                        model = message.content,
                        contentDescription = "Image message",
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(200.dp)
                            .clip(RoundedCornerShape(8.dp)),
                        contentScale = ContentScale.Crop
                    )
                }
                MessageType.VOICE -> {
                    VoiceMessagePlayer(message.content)
                }
            }
            
            Spacer(modifier = Modifier.height(4.dp))
            
            Row(
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text(
                    text = formatTime(message.timestamp),
                    style = MaterialTheme.typography.caption,
                    color = if (isCurrentUser) Color.White.copy(alpha = 0.7f) else Color.Gray
                )
                
                if (isCurrentUser) {
                    Icon(
                        imageVector = if (message.isRead) Icons.Default.DoneAll else Icons.Default.Done,
                        contentDescription = "Message status",
                        tint = Color.White.copy(alpha = 0.7f),
                        modifier = Modifier.size(16.dp)
                    )
                }
            }
        }
    }
}

@Composable
fun MessageInput(
    messageText: String,
    onMessageChange: (String) -> Unit,
    onSendMessage: (String, MessageType) -> Unit,
    isRecording: Boolean,
    onStartRecording: () -> Unit,
    onStopRecording: () -> Unit,
    isOnline: Boolean
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // Attachment button
        IconButton(onClick = { /* Show attachment options */ }) {
            Icon(
                imageVector = Icons.Default.AttachFile,
                contentDescription = "Attach file"
            )
        }
        
        // Text input or recording interface
        if (isRecording) {
            Row(
                verticalAlignment = Alignment.CenterVertically,
                modifier = Modifier
                    .weight(1f)
                    .background(Color.Red.copy(alpha = 0.1f), CircleShape)
                    .padding(12.dp)
            ) {
                Icon(
                    imageVector = Icons.Default.Mic,
                    contentDescription = "Recording",
                    tint = Color.Red
                )
                Spacer(modifier = Modifier.width(8.dp))
                Text(
                    text = "Recording... Tap to stop",
                    style = MaterialTheme.typography.body2,
                    color = Color.Red
                )
            }
            
            IconButton(onClick = onStopRecording) {
                Icon(
                    imageVector = Icons.Default.Stop,
                    contentDescription = "Stop recording",
                    tint = Color.Red
                )
            }
        } else {
            OutlinedTextField(
                value = messageText,
                onValueChange = onMessageChange,
                modifier = Modifier
                    .weight(1f)
                    .height(56.dp),
                placeholder = { Text("Type a message...") },
                maxLines = 3,
                singleLine = false
            )
            
            Spacer(modifier = Modifier.width(8.dp))
            
            if (messageText.isNotEmpty()) {
                // Send button
                IconButton(
                    onClick = { onSendMessage(messageText, MessageType.TEXT) },
                    enabled = isOnline
                ) {
                    Icon(
                        imageVector = Icons.Default.Send,
                        contentDescription = "Send message",
                        tint = if (isOnline) MaterialTheme.colors.primary else Color.Gray
                    )
                }
            } else {
                // Voice recording button
                IconButton(
                    onClick = onStartRecording,
                    enabled = isOnline
                ) {
                    Icon(
                        imageVector = Icons.Default.Mic,
                        contentDescription = "Record voice",
                        tint = if (isOnline) MaterialTheme.colors.primary else Color.Gray
                    )
                }
            }
        }
    }
}

@Composable
fun ChatRoomHeader(
    chatRoom: ChatRoom,
    isOnline: Boolean,
    typingUsers: Set<String>,
    onBack: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        IconButton(onClick = onBack) {
            Icon(
                imageVector = Icons.Default.ArrowBack,
                contentDescription = "Back"
            )
        }
        
        AsyncImage(
            model = chatRoom.avatarUrl,
            contentDescription = "Chat avatar",
            modifier = Modifier
                .size(40.dp)
                .clip(CircleShape),
            contentScale = ContentScale.Crop
        )
        
        Spacer(modifier = Modifier.width(12.dp))
        
        Column(modifier = Modifier.weight(1f)) {
            Text(
                text = chatRoom.name,
                style = MaterialTheme.typography.subtitle1
            )
            Text(
                text = when {
                    typingUsers.isNotEmpty() -> "Someone is typing..."
                    isOnline -> "Online" 
                    else -> "Last seen recently"
                },
                style = MaterialTheme.typography.caption,
                color = Color.Gray
            )
        }
        
        // Online status indicator
        Box(
            modifier = Modifier
                .size(12.dp)
                .background(
                    color = if (isOnline) Color.Green else Color.Gray,
                    shape = CircleShape
                )
        )
    }
}

// Supporting data classes
data class ChatRoom(
    val id: String,
    val name: String,
    val participants: List<User>,
    val avatarUrl: String?,
    val type: ChatType,
    val lastMessage: Message?,
    val unreadCount: Int
)

data class Message(
    val id: String,
    val content: String,
    val type: MessageType,
    val senderId: String,
    val senderName: String,
    val timestamp: Long,
    val chatRoomId: String,
    val isRead: Boolean,
    val replyToId: String? = null
)

enum class ChatType { DIRECT, GROUP }
enum class MessageType { TEXT, IMAGE, VOICE, FILE }

// Repository
object ChatRepository {
    private val messageListeners = mutableMapOf<String, (Message) -> Unit>()
    private val typingListeners = mutableMapOf<String, (String, Boolean) -> Unit>()
    
    suspend fun getMessages(chatRoomId: String): List<Message> {
        delay(200)
        return emptyList()
    }
    
    fun subscribeToMessages(chatRoomId: String, onMessage: (Message) -> Unit) {
        messageListeners[chatRoomId] = onMessage
        // Start listening to real-time messages
    }
    
    fun unsubscribeFromMessages(chatRoomId: String) {
        messageListeners.remove(chatRoomId)
    }
    
    fun subscribeToTyping(chatRoomId: String, onTyping: (String, Boolean) -> Unit) {
        typingListeners[chatRoomId] = onTyping
    }
    
    fun unsubscribeFromTyping(chatRoomId: String) {
        typingListeners.remove(chatRoomId)
    }
    
    suspend fun sendTyping(chatRoomId: String, userId: String, isTyping: Boolean) {
        // Send typing indicator to server
        typingListeners[chatRoomId]?.invoke(userId, isTyping)
    }
    
    suspend fun sendMessage(message: Message) {
        delay(100)
        messageListeners[message.chatRoomId]?.invoke(message)
    }
}

object UserManager {
    suspend fun getCurrentUser(): User {
        delay(200)
        return User("1", "Current User", "user@example.com")
    }
}

object NetworkManager {
    fun isConnected(): Boolean = true
}

suspend fun reconnectToChat() {
    // Implement reconnection logic
    delay(1000)
}

// Time formatting
@Composable
fun formatTime(timestamp: Long): String {
    return remember(timestamp) {
        val time = Calendar.getInstance().apply {
            timeInMillis = timestamp
        }
        SimpleDateFormat("HH:mm", Locale.getDefault()).format(time.time)
    }
}
```

### Application 3: E-commerce Shopping App

Let's create a comprehensive e-commerce application with product catalog, cart management, and payment processing:

```kotlin
@Composable
fun ECommerceApp() {
    var currentUser by remember { mutableStateOf<User?>(null) }
    var currentScreen by remember { mutableStateOf(AppScreen.HOME) }
    var cartItemCount by remember { mutableStateOf(0) }
    var isLoading by remember { mutableStateOf(true) }
    val scope = rememberCoroutineScope()
    
    // App initialization
    LaunchedEffect(Unit) {
        try {
            currentUser = UserManager.getCurrentUser()
            cartItemCount = CartManager.getCartItemCount()
            isLoading = false
        } catch (e: Exception) {
            Log.e("App", "Failed to initialize", e)
            isLoading = false
        }
    }
    
    // Cart sync effect
    LaunchedEffect(currentUser) {
        currentUser?.let { user ->
            CartManager.syncCart(user.id)
            cartItemCount = CartManager.getCartItemCount()
        }
    }
    
    when {
        isLoading -> ECommerceLoadingScreen()
        currentUser == null -> ECommerceAuthScreen(
            onLogin = { email, password ->
                scope.launch {
                    try {
                        isLoading = true
                        currentUser = UserManager.login(email, password)
                    } catch (e: Exception) {
                        // Handle login error
                    } finally {
                        isLoading = false
                    }
                }
            }
        )
        else -> ECommerceMainScreen(
            currentUser = currentUser!!,
            currentScreen = currentScreen,
            cartItemCount = cartItemCount,
            onScreenChange = { currentScreen = it },
            onCartUpdate = { cartItemCount = it }
        )
    }
}

@Composable
fun ECommerceMainScreen(
    currentUser: User,
    currentScreen: AppScreen,
    cartItemCount: Int,
    onScreenChange: (AppScreen) -> Unit,
    onCartUpdate: (Int) -> Unit
) {
    var isNetworkAvailable by remember { mutableStateOf(true) }
    val scope = rememberCoroutineScope()
    
    // Network monitoring
    DisposableEffect(Unit) {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                isNetworkAvailable = true
                scope.launch { handleNetworkRestore() }
            }
            
            override fun onLost(network: Network) {
                isNetworkAvailable = false
            }
        }
        
        connectivityManager.registerDefaultNetworkCallback(networkCallback)
        
        onDispose {
            connectivityManager.unregisterNetworkCallback(networkCallback)
        }
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        when (currentScreen) {
            AppScreen.HOME -> ProductCatalogScreen(
                currentUser = currentUser,
                onProductSelected = { /* Navigate to product detail */ },
                onAddToCart = { product ->
                    scope.launch {
                        CartManager.addToCart(currentUser.id, product)
                        onCartUpdate(CartManager.getCartItemCount())
                    }
                }
            )
            AppScreen.CART -> CartScreen(
                currentUser = currentUser,
                onCheckout = { /* Navigate to checkout */ },
                onUpdateQuantity = { productId, quantity ->
                    scope.launch {
                        CartManager.updateQuantity(currentUser.id, productId, quantity)
                        onCartUpdate(CartManager.getCartItemCount())
                    }
                },
                onRemoveFromCart = { productId ->
                    scope.launch {
                        CartManager.removeFromCart(currentUser.id, productId)
                        onCartUpdate(CartManager.getCartItemCount())
                    }
                }
            )
            AppScreen.ORDERS -> OrderHistoryScreen(currentUser = currentUser)
            AppScreen.PROFILE -> UserProfileScreen(currentUser = currentUser)
        }
        
        // Bottom navigation
        ECommerceBottomNavigation(
            currentScreen = currentScreen,
            cartItemCount = cartItemCount,
            onScreenChange = onScreenChange,
            modifier = Modifier.align(Alignment.BottomCenter)
        )
        
        // Network status banner
        if (!isNetworkAvailable) {
            NetworkStatusBanner(
                modifier = Modifier
                    .align(Alignment.TopCenter)
                    .fillMaxWidth()
            )
        }
    }
}

@Composable
fun ProductCatalogScreen(
    currentUser: User,
    onProductSelected: (Product) -> Unit,
    onAddToCart: (Product) -> Unit
) {
    var products by remember { mutableStateOf<List<Product>>(emptyList()) }
    var isLoading by remember { mutableStateOf(true) }
    var searchQuery by remember { mutableStateOf("") }
    var selectedCategory by remember { mutableStateOf<ProductCategory?>(null) }
    var sortBy by remember { mutableStateOf(ProductSort.POPULARITY) }
    val scope = rememberCoroutineScope()
    
    // Load products effect
    LaunchedEffect(Unit) {
        try {
            products = withContext(Dispatchers.IO) {
                ProductRepository.getProducts()
            }
            isLoading = false
        } catch (e: Exception) {
            Log.e("ProductCatalog", "Failed to load products", e)
            isLoading = false
        }
    }
    
    // Real-time product updates
    LaunchedEffect(Unit) {
        ProductRepository.subscribeToProductUpdates { updatedProduct ->
            products = products.map { 
                if (it.id == updatedProduct.id) updatedProduct else it
            }
        }
    }
    
    // Filter and sort products
    val filteredProducts = remember(products, searchQuery, selectedCategory, sortBy) {
        var result = products
        
        // Apply search
        if (searchQuery.isNotEmpty()) {
            result = result.filter { 
                it.name.contains(searchQuery, ignoreCase = true) ||
                it.description.contains(searchQuery, ignoreCase = true)
            }
        }
        
        // Apply category filter
        selectedCategory?.let { category ->
            result = result.filter { it.category == category }
        }
        
        // Apply sorting
        result = when (sortBy) {
            ProductSort.POPULARITY -> result.sortedByDescending { it.popularity }
            ProductSort.PRICE_LOW_TO_HIGH -> result.sortedBy { it.price }
            ProductSort.PRICE_HIGH_TO_LOW -> result.sortedByDescending { it.price }
            ProductSort.RATING -> result.sortedByDescending { it.rating }
            ProductSort.NEWEST -> result.sortedByDescending { it.createdAt }
        }
        
        result
    }
    
    // Analytics tracking
    SideEffect {
        Analytics.trackEvent("product_catalog_viewed", mapOf(
            "user_id" to currentUser.id,
            "product_count" to products.size,
            "filter_applied" to (selectedCategory != null),
            "search_query" to searchQuery
        ))
    }
    
    Column {
        // Search and filter header
        ProductCatalogHeader(
            searchQuery = searchQuery,
            onSearchChange = { searchQuery = it },
            selectedCategory = selectedCategory,
            onCategoryChange = { selectedCategory = it },
            sortBy = sortBy,
            onSortChange = { sortBy = it }
        )
        
        // Products grid
        when {
            isLoading -> ProductCatalogLoading()
            filteredProducts.isEmpty() -> ProductCatalogEmpty()
            else -> LazyVerticalGrid(
                columns = GridCells.Fixed(2),
                contentPadding = PaddingValues(16.dp)
            ) {
                items(filteredProducts) { product ->
                    ProductCard(
                        product = product,
                        onProductClick = { onProductSelected(product) },
                        onAddToCart = { onAddToCart(product) },
                        isInCart = CartManager.isInCart(currentUser.id, product.id)
                    )
                }
            }
        }
    }
}

@Composable
fun ProductCard(
    product: Product,
    onProductClick: () -> Unit,
    onAddToCart: () -> Unit,
    isInCart: Boolean
) {
    var isAddingToCart by remember { mutableStateOf(false) }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(4.dp)
            .clickable { onProductClick() },
        elevation = 4.dp
    ) {
        Column {
            // Product image
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(150.dp),
                contentScale = ContentScale.Crop
            )
            
            Column(modifier = Modifier.padding(12.dp)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.subtitle1,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                
                Spacer(modifier = Modifier.height(4.dp))
                
                // Rating
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(
                        imageVector = Icons.Default.Star,
                        contentDescription = "Rating",
                        tint = Color.Yellow,
                        modifier = Modifier.size(16.dp)
                    )
                    Text(
                        text = "${product.rating} (${product.reviewCount})",
                        style = MaterialTheme.typography.caption,
                        color = Color.Gray
                    )
                }
                
                Spacer(modifier = Modifier.height(8.dp))
                
                // Price
                Text(
                    text = "$${product.price}",
                    style = MaterialTheme.typography.h6,
                    color = MaterialTheme.colors.primary
                )
                
                // Discount badge
                if (product.originalPrice > product.price) {
                    Row {
                        Text(
                            text = "$${product.originalPrice}",
                            style = MaterialTheme.typography.caption,
                            color = Color.Gray,
                            textDecoration = TextDecoration.LineThrough
                        )
                        Spacer(modifier = Modifier.width(4.dp))
                        Text(
                            text = "${calculateDiscount(product.originalPrice, product.price)}% off",
                            style = MaterialTheme.typography.caption,
                            color = Color.Green
                        )
                    }
                }
                
                Spacer(modifier = Modifier.height(8.dp))
                
                // Add to cart button
                Button(
                    onClick = {
                        if (!isInCart) {
                            isAddingToCart = true
                            onAddToCart()
                            // Simulate add to cart delay
                            scope.launch {
                                delay(500)
                                isAddingToCart = false
                            }
                        }
                    },
                    enabled = !isAddingToCart,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    if (isAddingToCart) {
                        CircularProgressIndicator(
                            modifier = Modifier.size(16.dp),
                            color = Color.White
                        )
                    } else {
                        Text(if (isInCart) "In Cart" else "Add to Cart")
                    }
                }
            }
        }
    }
}

@Composable
fun CartScreen(
    currentUser: User,
    onCheckout: () -> Unit,
    onUpdateQuantity: (String, Int) -> Unit,
    onRemoveFromCart: (String) -> Unit
) {
    var cartItems by remember { mutableStateOf<List<CartItem>>(emptyList()) }
    var isLoading by remember { mutableStateOf(true) }
    var totalAmount by remember { mutableStateOf(0.0) }
    var shippingCost by remember { mutableStateOf(0.0) }
    var tax by remember { mutableStateOf(0.0) }
    val scope = rememberCoroutineScope()
    
    // Load cart items
    LaunchedEffect(currentUser) {
        try {
            cartItems = withContext(Dispatchers.IO) {
                CartManager.getCartItems(currentUser.id)
            }
            calculateTotals()
            isLoading = false
        } catch (e: Exception) {
            Log.e("Cart", "Failed to load cart", e)
            isLoading = false
        }
    }
    
    // Real-time cart updates
    LaunchedEffect(currentUser.id) {
        CartManager.subscribeToCartUpdates(currentUser.id) { cartItem ->
            cartItems = cartItems.map { 
                if (it.productId == cartItem.productId) cartItem else it
            }
            calculateTotals()
        }
    }
    
    // Cart cleanup effect
    DisposableEffect(Unit) {
        onDispose {
            CartManager.unsubscribeFromCartUpdates(currentUser.id)
        }
    }
    
    // Calculate totals effect
    fun calculateTotals() {
        val itemsTotal = cartItems.sumOf { it.product.price * it.quantity }
        shippingCost = if (itemsTotal > 50) 0.0 else 5.0
        tax = itemsTotal * 0.08 // 8% tax
        totalAmount = itemsTotal + shippingCost + tax
    }
    
    when {
        isLoading -> CartLoadingScreen()
        cartItems.isEmpty() -> CartEmptyScreen()
        else -> Column {
            // Cart items
            LazyColumn(
                modifier = Modifier
                    .weight(1f)
                    .fillMaxWidth()
            ) {
                items(cartItems) { cartItem ->
                    CartItemRow(
                        cartItem = cartItem,
                        onUpdateQuantity = { quantity ->
                            onUpdateQuantity(cartItem.productId, quantity)
                        },
                        onRemove = { onRemoveFromCart(cartItem.productId) }
                    )
                }
            }
            
            // Order summary
            OrderSummaryCard(
                itemsTotal = cartItems.sumOf { it.product.price * it.quantity },
                shippingCost = shippingCost,
                tax = tax,
                totalAmount = totalAmount,
                onCheckout = onCheckout
            )
        }
    }
}

@Composable
fun CartItemRow(
    cartItem: CartItem,
    onUpdateQuantity: (Int) -> Unit,
    onRemove: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // Product image
        AsyncImage(
            model = cartItem.product.imageUrl,
            contentDescription = cartItem.product.name,
            modifier = Modifier
                .size(80.dp)
                .clip(RoundedCornerShape(8.dp)),
            contentScale = ContentScale.Crop
        )
        
        Spacer(modifier = Modifier.width(12.dp))
        
        // Product details
        Column(modifier = Modifier.weight(1f)) {
            Text(
                text = cartItem.product.name,
                style = MaterialTheme.typography.subtitle1,
                maxLines = 2,
                overflow = TextOverflow.Ellipsis
            )
            
            Text(
                text = "$${cartItem.product.price} each",
                style = MaterialTheme.typography.body2,
                color = Color.Gray
            )
            
            Spacer(modifier = Modifier.height(8.dp))
            
            // Quantity controls
            Row(
                verticalAlignment = Alignment.CenterVertically
            ) {
                IconButton(onClick = { 
                    if (cartItem.quantity > 1) {
                        onUpdateQuantity(cartItem.quantity - 1)
                    }
                }) {
                    Icon(
                        imageVector = Icons.Default.Remove,
                        contentDescription = "Decrease quantity"
                    )
                }
                
                Text(
                    text = cartItem.quantity.toString(),
                    style = MaterialTheme.typography.body1,
                    modifier = Modifier.padding(horizontal = 16.dp)
                )
                
                IconButton(onClick = { 
                    onUpdateQuantity(cartItem.quantity + 1)
                }) {
                    Icon(
                        imageVector = Icons.Default.Add,
                        contentDescription = "Increase quantity"
                    )
                }
            }
        }
        
        Column(horizontalAlignment = Alignment.End) {
            Text(
                text = "$${cartItem.product.price * cartItem.quantity}",
                style = MaterialTheme.typography.subtitle1
            )
            
            IconButton(onClick = onRemove) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = "Remove item",
                    tint = Color.Red
                )
            }
        }
    }
}

@Composable
fun OrderSummaryCard(
    itemsTotal: Double,
    shippingCost: Double,
    tax: Double,
    totalAmount: Double,
    onCheckout: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        elevation = 8.dp
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = "Order Summary",
                style = MaterialTheme.typography.h6
            )
            
            Spacer(modifier = Modifier.height(8.dp))
            
            // Items total
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text("Items Total:")
                Text("$${String.format("%.2f", itemsTotal)}")
            }
            
            // Shipping
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text("Shipping:")
                Text(
                    text = if (shippingCost > 0) "$${String.format("%.2f", shippingCost)}" else "FREE",
                    color = if (shippingCost > 0) Color.Black else Color.Green
                )
            }
            
            // Tax
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text("Tax:")
                Text("$${String.format("%.2f", tax)}")
            }
            
            Divider(modifier = Modifier.padding(vertical = 8.dp))
            
            // Total
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text(
                    text = "Total:",
                    style = MaterialTheme.typography.h6
                )
                Text(
                    text = "$${String.format("%.2f", totalAmount)}",
                    style = MaterialTheme.typography.h6,
                    color = MaterialTheme.colors.primary
                )
            }
            
            if (itemsTotal < 50) {
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = "Add $${String.format("%.2f", 50 - itemsTotal)} more for free shipping!",
                    style = MaterialTheme.typography.caption,
                    color = Color.Green
                )
            }
            
            Spacer(modifier = Modifier.height(16.dp))
            
            // Checkout button
            Button(
                onClick = onCheckout,
                modifier = Modifier.fillMaxWidth(),
                enabled = totalAmount > 0
            ) {
                Text("Proceed to Checkout")
            }
        }
    }
}

@Composable
fun ECommerceBottomNavigation(
    currentScreen: AppScreen,
    cartItemCount: Int,
    onScreenChange: (AppScreen) -> Unit,
    modifier: Modifier = Modifier
) {
    NavigationBar(modifier = modifier) {
        NavigationBarItem(
            icon = { Icon(Icons.Default.Home, contentDescription = "Home") },
            label = { Text("Home") },
            selected = currentScreen == AppScreen.HOME,
            onClick = { onScreenChange(AppScreen.HOME) }
        )
        
        NavigationBarItem(
            icon = { 
                Box {
                    Icon(Icons.Default.ShoppingCart, contentDescription = "Cart")
                    if (cartItemCount > 0) {
                        Badge(
                            boxContentColor = Color.White,
                            modifier = Modifier.align(Alignment.TopEnd)
                        ) {
                            Text(cartItemCount.toString())
                        }
                    }
                }
            },
            label = { Text("Cart") },
            selected = currentScreen == AppScreen.CART,
            onClick = { onScreenChange(AppScreen.CART) }
        )
        
        NavigationBarItem(
            icon = { Icon(Icons.Default.Receipt, contentDescription = "Orders") },
            label = { Text("Orders") },
            selected = currentScreen == AppScreen.ORDERS,
            onClick = { onScreenChange(AppScreen.ORDERS) }
        )
        
        NavigationBarItem(
            icon = { Icon(Icons.Default.Person, contentDescription = "Profile") },
            label = { Text("Profile") },
            selected = currentScreen == AppScreen.PROFILE,
            onClick = { onScreenChange(AppScreen.PROFILE) }
        )
    }
}

@Composable
fun NetworkStatusBanner(
    modifier: Modifier = Modifier
) {
    Row(
        modifier = modifier
            .background(Color.Red.copy(alpha = 0.8f))
            .padding(16.dp),
        horizontalArrangement = Arrangement.Center,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(
            imageVector = Icons.Default.WifiOff,
            contentDescription = "No internet",
            tint = Color.White
        )
        Spacer(modifier = Modifier.width(8.dp))
        Text(
            text = "No internet connection",
            color = Color.White,
            style = MaterialTheme.typography.body2
        )
    }
}

// Supporting data classes
enum class AppScreen { HOME, CART, ORDERS, PROFILE }

data class Product(
    val id: String,
    val name: String,
    val description: String,
    val price: Double,
    val originalPrice: Double,
    val imageUrl: String,
    val category: ProductCategory,
    val rating: Double,
    val reviewCount: Int,
    val popularity: Int,
    val createdAt: Long,
    val isAvailable: Boolean,
    val stockCount: Int
)

data class CartItem(
    val productId: String,
    val quantity: Int,
    val addedAt: Long,
    val product: Product
)

enum class ProductCategory { ELECTRONICS, CLOTHING, BOOKS, HOME, SPORTS }
enum class ProductSort { POPULARITY, PRICE_LOW_TO_HIGH, PRICE_HIGH_TO_LOW, RATING, NEWEST }

// Repository and services
object ProductRepository {
    suspend fun getProducts(): List<Product> {
        delay(300)
        return emptyList() // Return actual products
    }
    
    fun subscribeToProductUpdates(onUpdate: (Product) -> Unit) {
        // Subscribe to real-time product updates
    }
}

object CartManager {
    private val cartListeners = mutableMapOf<String, (CartItem) -> Unit>()
    
    suspend fun getCartItems(userId: String): List<CartItem> {
        delay(200)
        return emptyList()
    }
    
    suspend fun getCartItemCount(): Int {
        delay(100)
        return 0
    }
    
    suspend fun addToCart(userId: String, product: Product) {
        delay(100)
        // Add to cart logic
    }
    
    suspend fun updateQuantity(userId: String, productId: String, quantity: Int) {
        delay(100)
        // Update quantity logic
    }
    
    suspend fun removeFromCart(userId: String, productId: String) {
        delay(100)
        // Remove from cart logic
    }
    
    suspend fun syncCart(userId: String) {
        delay(200)
        // Sync with server
    }
    
    fun isInCart(userId: String, productId: String): Boolean {
        return false // Check if in cart
    }
    
    fun subscribeToCartUpdates(userId: String, onUpdate: (CartItem) -> Unit) {
        cartListeners[userId] = onUpdate
    }
    
    fun unsubscribeFromCartUpdates(userId: String) {
        cartListeners.remove(userId)
    }
}

// Utility functions
fun calculateDiscount(originalPrice: Double, currentPrice: Double): Int {
    return ((originalPrice - currentPrice) / originalPrice * 100).toInt()
}
```

### Key Takeaways from Real-world Applications

1. **Integration is Key**: Real applications combine multiple side effects seamlessly
2. **State Management**: Proper state hoisting and management is crucial
3. **Real-time Updates**: Effects enable real-time data synchronization
4. **Network Handling**: Monitor connectivity and handle offline scenarios
5. **User Experience**: Loading states, error handling, and feedback are essential
6. **Performance**: Optimize for large datasets and complex interactions
7. **Analytics**: Track user behavior and app performance
8. **Scalability**: Design patterns that work with growing complexity

### Best Practices for Production Apps

1. **Error Boundaries**: Implement comprehensive error handling
2. **Offline Support**: Design for intermittent connectivity
3. **Performance Monitoring**: Track app performance and user experience
4. **Security**: Protect user data and implement proper authentication
5. **Testing**: Write comprehensive tests for all side effect patterns
6. **Documentation**: Document complex patterns and architecture decisions
7. **Code Organization**: Maintain clean separation of concerns
8. **Resource Management**: Ensure proper cleanup of resources

This completes our comprehensive guide to Jetpack Compose side effects! You now have all the knowledge and practical examples needed to build production-ready applications using proper side effect management in Compose.