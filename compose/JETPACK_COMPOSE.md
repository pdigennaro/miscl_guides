Great! Let’s begin with **Module 1: Introduction & Setup**.

---

## ✅ Module 1: Introduction & Setup

### 🎯 **Goal:** Set up your environment and understand how Jetpack Compose differs from XML.

---

### 1. **Install Android Studio**

* Download the latest version: [https://developer.android.com/studio](https://developer.android.com/studio)
* Install using standard options for your platform.
* Ensure **Kotlin Plugin** is installed (it usually is by default).

---

### 2. **Create Your First Compose Project**

**Steps:**

1. Open Android Studio → New Project → Select **“Empty Compose Activity”**.
2. Name: `HelloCompose`
3. Minimum SDK: API 21 or higher
4. Finish.

**What you’ll see:**

* `MainActivity.kt` using `setContent {}` to set the Compose UI.
* `@Composable` annotated function (e.g., `Greeting()`).

---

### 3. **Understand the Basics**

| XML UI (Legacy)         | Jetpack Compose (Modern)   |
| ----------------------- | -------------------------- |
| Imperative UI updates   | Declarative UI             |
| Separate layout XML     | UI is built in Kotlin code |
| `findViewById()`        | Direct use of composables  |
| UI = data binding extra | UI = function of state     |

---

### 4. **Your First App: Hello Compose**

Paste this into your `MainActivity.kt`:

```kotlin
@Composable
fun HelloText() {
    Text(text = "Hello Jetpack Compose!", fontSize = 24.sp)
}

@Preview(showBackground = true)
@Composable
fun HelloTextPreview() {
    HelloText()
}
```

Update `onCreate()`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent {
        HelloText()
    }
}
```

✅ Run your app — you should see the message on screen.

---

### 🧪 Challenge Exercise

1. Create another composable that shows your name and profession.
2. Add some styling using font size and color.
3. Display both in a `Column`.

---

## ✅ Module 2: Basic Composables

### 🎯 **Goal:** Learn the building blocks of Compose UI — composable functions, text, buttons, and layouts.

---

### 1. **What is a @Composable Function?**

* Annotated with `@Composable`
* Defines part of your UI
* Can call other composables

Example:

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

---

### 2. **Text & Typography**

```kotlin
@Composable
fun CustomText() {
    Text(
        text = "Jetpack Compose Rocks!",
        fontSize = 20.sp,
        fontWeight = FontWeight.Bold,
        color = Color.Blue
    )
}
```

---

### 3. **Button**

```kotlin
@Composable
fun ClickMeButton() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("Clicked $count times")
    }
}
```

---

### 4. **Layout: Column, Row, Box**

```kotlin
@Composable
fun SimpleLayout() {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Name: Alex")
        Text("Occupation: Android Developer")
        ClickMeButton()
    }
}
```

---

### 5. **Previewing UI**

Use the `@Preview` annotation to visualize your UI without launching the emulator:

```kotlin
@Preview(showBackground = true)
@Composable
fun PreviewLayout() {
    SimpleLayout()
}
```

---

### 🧪 Practice Challenge

Build a **Profile Card UI** that contains:

* Your avatar (use `Image`)
* Your name (`Text`)
* A short description (`Text`)
* A button to follow (`Button`)

Hint: Use `Column` and `Modifier.padding()`, with optional styling.

---

## 🧑‍💼 Profile Card UI – Example

### 📦 Requirements:

* Avatar image (you can use a placeholder from resources)
* Name and role
* Short description
* Follow button

### 🧩 Code:

```kotlin
@Composable
fun ProfileCard() {
    Card(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 8.dp),
        shape = RoundedCornerShape(12.dp)
    ) {
        Column(
            modifier = Modifier
                .padding(16.dp)
                .fillMaxWidth(),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            // Avatar Image (from drawable)
            Image(
                painter = painterResource(id = R.drawable.ic_launcher_foreground),
                contentDescription = "Profile Picture",
                modifier = Modifier
                    .size(100.dp)
                    .clip(CircleShape)
            )

            Spacer(modifier = Modifier.height(12.dp))

            // Name
            Text(
                text = "Alex Johnson",
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold
            )

            // Role
            Text(
                text = "Android Developer",
                fontSize = 16.sp,
                color = Color.Gray
            )

            Spacer(modifier = Modifier.height(8.dp))

            // Description
            Text(
                text = "Passionate about Kotlin, Compose, and clean architecture.",
                textAlign = TextAlign.Center,
                modifier = Modifier.padding(horizontal = 16.dp)
            )

            Spacer(modifier = Modifier.height(12.dp))

            // Follow Button
            var isFollowing by remember { mutableStateOf(false) }

            Button(
                onClick = { isFollowing = !isFollowing }
            ) {
                Text(if (isFollowing) "Unfollow" else "Follow")
            }
        }
    }
}
```

### 🔍 Add Preview

```kotlin
@Preview(showBackground = true)
@Composable
fun ProfileCardPreview() {
    ProfileCard()
}
```

---

### 📝 Notes:

* Replace `R.drawable.ic_launcher_foreground` with your own image in `res/drawable` (e.g., `profile.png`).
* You can enhance this with a `Row` layout if you want the avatar on the left.

---

## ✅ Module 3: Layouts & Modifiers

### 🎯 **Goal:** Structure your UI with layout composables and style it using modifiers.

---

## 📦 Core Concepts

### 1. **Layout Containers**

| Layout     | Purpose                             |
| ---------- | ----------------------------------- |
| `Column`   | Stack children vertically           |
| `Row`      | Stack children horizontally         |
| `Box`      | Stack children on top of each other |
| `Scaffold` | App layout structure (top bar, fab) |

---

### 2. **Using Modifiers**

`Modifier` is used to style, size, and control layout behavior.

#### Common Modifiers:

* `padding()`
* `fillMaxWidth()`, `fillMaxSize()`, `width()`, `height()`
* `background()`, `clip()`
* `clickable()`
* `align()`, `wrapContentSize()`

---

## 🧩 Example: Responsive Blog Post Layout

```kotlin
@Composable
fun BlogPostCard(title: String, author: String, body: String) {
    Card(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = title,
                style = MaterialTheme.typography.titleLarge
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = "By $author",
                style = MaterialTheme.typography.bodySmall,
                color = Color.Gray
            )
            Spacer(modifier = Modifier.height(12.dp))
            Text(
                text = body,
                style = MaterialTheme.typography.bodyMedium,
                maxLines = 3,
                overflow = TextOverflow.Ellipsis
            )
        }
    }
}
```

### 📱 Preview

```kotlin
@Preview(showBackground = true)
@Composable
fun BlogPostPreview() {
    BlogPostCard(
        title = "Mastering Jetpack Compose",
        author = "Jane Doe",
        body = "Jetpack Compose is Android’s modern toolkit for building native UI. It's powerful, declarative, and built entirely in Kotlin..."
    )
}
```

---

## 🧪 Challenge: Layout Practice

Build a **Login Screen** using:

* `Column` for vertical layout
* `TextField` for email & password
* A `Button` to log in
* Center everything using `Modifier.fillMaxSize().wrapContentSize(Alignment.Center)`

---

## 🔐 Login Screen UI

### 📦 Features:

* Email & Password input fields
* Login button
* Basic input state handling
* Centered vertical layout

---

### ✅ Code:

```kotlin
@Composable
fun LoginScreen() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp)
            .wrapContentSize(Alignment.Center),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Login",
            fontSize = 28.sp,
            fontWeight = FontWeight.Bold,
            modifier = Modifier.padding(bottom = 24.dp)
        )

        // Email Field
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") },
            singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(16.dp))

        // Password Field
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            singleLine = true,
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(24.dp))

        // Login Button
        Button(
            onClick = { /* You can add validation or nav logic here */ },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Log In")
        }
    }
}
```

---

### 📱 Preview

```kotlin
@Preview(showBackground = true)
@Composable
fun LoginScreenPreview() {
    LoginScreen()
}
```

---

### 📝 Notes:

* `remember` stores the current input state during recomposition.
* `PasswordVisualTransformation()` hides the password.
* You can add validation, navigation, or snackbar feedback later.

---

## ✅ Module 4: State & User Input

### 🎯 **Goal:** Understand how to manage state and handle user interaction in a declarative UI model.

---

## 📦 Core Concepts

### 1. **State in Compose**

Jetpack Compose is declarative. This means:

* You describe *what* the UI should look like **based on state**.
* When the state changes, the UI is **automatically redrawn**.

### 2. **Using `remember` and `mutableStateOf`**

```kotlin
var name by remember { mutableStateOf("") }
```

This creates a state variable that survives recompositions.

* `remember` – stores the value in memory
* `mutableStateOf` – wraps the value in a state holder
* `by` – Kotlin property delegate to auto-unpack

---

### 3. **Handling Input (TextField)**

```kotlin
OutlinedTextField(
    value = name,
    onValueChange = { name = it },
    label = { Text("Name") }
)
```

When the user types, the lambda updates `name`, which triggers UI to update.

---

### 🔁 Example: Realtime Greeting App

```kotlin
@Composable
fun GreetingInput() {
    var name by remember { mutableStateOf("") }

    Column(
        modifier = Modifier
            .padding(24.dp)
            .fillMaxSize()
            .wrapContentSize(Alignment.Center),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Enter your name") }
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text(
            text = if (name.isNotBlank()) "Hello, $name!" else "Please enter your name.",
            fontSize = 20.sp,
            fontWeight = FontWeight.Medium
        )
    }
}
```

---

### 🧪 Challenge Exercise

Build a **feedback form** with:

* A name input
* A message input
* A submit button that, when clicked, shows a "Thank you!" message or Snackbar

---

## ✅ Feedback Form UI

### 📦 Features:

* Name input
* Feedback message input
* Submit button
* “Thank you!” message displayed conditionally

---

### ✨ Code:

```kotlin
@Composable
fun FeedbackForm() {
    var name by remember { mutableStateOf("") }
    var message by remember { mutableStateOf("") }
    var submitted by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier
            .padding(24.dp)
            .fillMaxSize()
            .wrapContentSize(Alignment.Center),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Feedback Form",
            fontSize = 24.sp,
            fontWeight = FontWeight.Bold,
            modifier = Modifier.padding(bottom = 16.dp)
        )

        OutlinedTextField(
            value = name,
            onValueChange = {
                name = it
                submitted = false
            },
            label = { Text("Your Name") },
            singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = message,
            onValueChange = {
                message = it
                submitted = false
            },
            label = { Text("Your Message") },
            modifier = Modifier
                .fillMaxWidth()
                .height(120.dp),
            maxLines = 5
        )

        Spacer(modifier = Modifier.height(24.dp))

        Button(
            onClick = {
                if (name.isNotBlank() && message.isNotBlank()) {
                    submitted = true
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Submit")
        }

        if (submitted) {
            Spacer(modifier = Modifier.height(24.dp))
            Text(
                text = "Thank you, $name!",
                color = Color(0xFF2E7D32),
                fontWeight = FontWeight.Medium,
                fontSize = 18.sp
            )
        }
    }
}
```

---

### 🖼 Preview:

```kotlin
@Preview(showBackground = true)
@Composable
fun FeedbackFormPreview() {
    FeedbackForm()
}
```

---

### 🧠 Notes:

* State is updated in real time as the user types.
* `submitted` toggles the confirmation message.
* You could also trigger a `Snackbar`, `Dialog`, or Toast here instead.

---

Great! Let’s enhance the **Feedback Form** by adding:

### ✅ 1. Form Validation

### ✅ 2. Snackbar for Submission Feedback

---

## 🔐 Updated Feedback Form with Validation & Snackbar

### ✨ Highlights:

* Shows validation errors if fields are blank
* Displays a Snackbar when submission is successful
* Uses `Scaffold` and `SnackbarHostState`

---

### 🔧 Full Code:

```kotlin
@Composable
fun FeedbackFormWithValidation() {
    val snackbarHostState = remember { SnackbarHostState() }
    val coroutineScope = rememberCoroutineScope()

    var name by remember { mutableStateOf("") }
    var message by remember { mutableStateOf("") }
    var nameError by remember { mutableStateOf(false) }
    var messageError by remember { mutableStateOf(false) }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) },
        modifier = Modifier.fillMaxSize()
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .padding(paddingValues)
                .padding(24.dp)
                .fillMaxSize()
                .wrapContentSize(Alignment.Center),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                text = "Feedback Form",
                fontSize = 24.sp,
                fontWeight = FontWeight.Bold,
                modifier = Modifier.padding(bottom = 16.dp)
            )

            OutlinedTextField(
                value = name,
                onValueChange = {
                    name = it
                    nameError = false
                },
                label = { Text("Your Name") },
                isError = nameError,
                singleLine = true,
                modifier = Modifier.fillMaxWidth()
            )

            if (nameError) {
                Text(
                    "Name cannot be empty",
                    color = Color.Red,
                    fontSize = 12.sp,
                    modifier = Modifier.align(Alignment.Start)
                )
            }

            Spacer(modifier = Modifier.height(16.dp))

            OutlinedTextField(
                value = message,
                onValueChange = {
                    message = it
                    messageError = false
                },
                label = { Text("Your Message") },
                isError = messageError,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(120.dp),
                maxLines = 5
            )

            if (messageError) {
                Text(
                    "Message cannot be empty",
                    color = Color.Red,
                    fontSize = 12.sp,
                    modifier = Modifier.align(Alignment.Start)
                )
            }

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = {
                    nameError = name.isBlank()
                    messageError = message.isBlank()

                    if (!nameError && !messageError) {
                        coroutineScope.launch {
                            snackbarHostState.showSnackbar("Thanks for your feedback, $name!")
                        }
                        name = ""
                        message = ""
                    }
                },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Submit")
            }
        }
    }
}
```

---

### 🖼 Preview:

```kotlin
@Preview(showBackground = true)
@Composable
fun FeedbackFormWithValidationPreview() {
    FeedbackFormWithValidation()
}
```

---

### 🧠 Key Concepts Used:

* **`isError = true`** in `TextField` for validation UI
* **`SnackbarHostState` + `CoroutineScope`** to show snackbars
* **`Scaffold`** wraps everything and places snackbar automatically

---

## ✨ Animate Validation Feedback in Compose

We’ll animate:

1. **Error text visibility** using `AnimatedVisibility`
2. **TextField border color changes** with smooth transition

---

### 🔧 Updated Code with Animations

```kotlin
@Composable
fun AnimatedFeedbackForm() {
    val snackbarHostState = remember { SnackbarHostState() }
    val coroutineScope = rememberCoroutineScope()

    var name by remember { mutableStateOf("") }
    var message by remember { mutableStateOf("") }
    var nameError by remember { mutableStateOf(false) }
    var messageError by remember { mutableStateOf(false) }

    val nameBorderColor by animateColorAsState(
        if (nameError) Color.Red else MaterialTheme.colorScheme.outline
    )

    val messageBorderColor by animateColorAsState(
        if (messageError) Color.Red else MaterialTheme.colorScheme.outline
    )

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) },
        modifier = Modifier.fillMaxSize()
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .padding(paddingValues)
                .padding(24.dp)
                .fillMaxSize()
                .wrapContentSize(Alignment.Center),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                text = "Feedback Form",
                fontSize = 24.sp,
                fontWeight = FontWeight.Bold,
                modifier = Modifier.padding(bottom = 16.dp)
            )

            OutlinedTextField(
                value = name,
                onValueChange = {
                    name = it
                    nameError = false
                },
                label = { Text("Your Name") },
                singleLine = true,
                isError = nameError,
                modifier = Modifier
                    .fillMaxWidth()
                    .border(1.dp, nameBorderColor, RoundedCornerShape(6.dp))
            )

            AnimatedVisibility(visible = nameError) {
                Text(
                    "Name cannot be empty",
                    color = Color.Red,
                    fontSize = 12.sp,
                    modifier = Modifier
                        .align(Alignment.Start)
                        .padding(top = 4.dp)
                )
            }

            Spacer(modifier = Modifier.height(16.dp))

            OutlinedTextField(
                value = message,
                onValueChange = {
                    message = it
                    messageError = false
                },
                label = { Text("Your Message") },
                isError = messageError,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(120.dp)
                    .border(1.dp, messageBorderColor, RoundedCornerShape(6.dp)),
                maxLines = 5
            )

            AnimatedVisibility(visible = messageError) {
                Text(
                    "Message cannot be empty",
                    color = Color.Red,
                    fontSize = 12.sp,
                    modifier = Modifier
                        .align(Alignment.Start)
                        .padding(top = 4.dp)
                )
            }

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = {
                    nameError = name.isBlank()
                    messageError = message.isBlank()

                    if (!nameError && !messageError) {
                        coroutineScope.launch {
                            snackbarHostState.showSnackbar("Thanks for your feedback, $name!")
                        }
                        name = ""
                        message = ""
                    }
                },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Submit")
            }
        }
    }
}
```

---

### 🎉 What's Animated:

* **Error messages** fade in/out smoothly using `AnimatedVisibility`
* **TextField borders** change color with a smooth transition using `animateColorAsState`

---

## 🧮 Add Input Length Limits to the Feedback Form

### 🎯 Goals:

* Limit **name** to 30 characters
* Limit **message** to 200 characters
* Show live **character counter**
* Prevent user from typing beyond the limit

---

### ✅ Updated Snippets (Name + Message Field):

```kotlin
// NAME INPUT
OutlinedTextField(
    value = name,
    onValueChange = {
        if (it.length <= 30) {
            name = it
            nameError = false
        }
    },
    label = { Text("Your Name") },
    singleLine = true,
    isError = nameError,
    modifier = Modifier
        .fillMaxWidth()
        .border(1.dp, nameBorderColor, RoundedCornerShape(6.dp)),
    supportingText = {
        Text("${name.length}/30")
    }
)

// MESSAGE INPUT
OutlinedTextField(
    value = message,
    onValueChange = {
        if (it.length <= 200) {
            message = it
            messageError = false
        }
    },
    label = { Text("Your Message") },
    isError = messageError,
    modifier = Modifier
        .fillMaxWidth()
        .height(120.dp)
        .border(1.dp, messageBorderColor, RoundedCornerShape(6.dp)),
    maxLines = 5,
    supportingText = {
        Text("${message.length}/200")
    }
)
```

---

### 🔍 What This Adds:

* Prevents typing beyond the max length
* Displays a counter like `12/30` or `143/200`
* All other features like validation and animation remain intact

---

## ✅ Format Validation in Compose

### 🎯 Goals:

* **Name** must contain only letters and spaces
* **Email** must match a valid email pattern
* Show **animated error messages**
* Highlight fields with **color animation**

---

### 📌 Add Email Field With Validation

First, add a regex pattern at the top of your composable:

```kotlin
val emailPattern = Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,6}$")
```

---

### 🧩 Updated State and Field Logic

Add email field with validation:

```kotlin
var email by remember { mutableStateOf("") }
var emailError by remember { mutableStateOf(false) }

val emailBorderColor by animateColorAsState(
    if (emailError) Color.Red else MaterialTheme.colorScheme.outline
)
```

Now insert the `OutlinedTextField`:

```kotlin
OutlinedTextField(
    value = email,
    onValueChange = {
        email = it
        emailError = false
    },
    label = { Text("Your Email") },
    singleLine = true,
    isError = emailError,
    modifier = Modifier
        .fillMaxWidth()
        .border(1.dp, emailBorderColor, RoundedCornerShape(6.dp)),
    supportingText = {
        Text("${email.length}/50")
    }
)

AnimatedVisibility(visible = emailError) {
    Text(
        "Enter a valid email",
        color = Color.Red,
        fontSize = 12.sp,
        modifier = Modifier.align(Alignment.Start).padding(top = 4.dp)
    )
}
```

---

### ✨ Name Validation (Only Letters and Spaces)

Replace the name’s `onClick` block with:

```kotlin
val namePattern = Regex("^[A-Za-z ]+$")

nameError = name.isBlank() || !name.matches(namePattern)
emailError = email.isBlank() || !email.matches(emailPattern)
messageError = message.isBlank()
```

---

### 🔁 Final Submit Check

```kotlin
if (!nameError && !emailError && !messageError) {
    coroutineScope.launch {
        snackbarHostState.showSnackbar("Thanks, $name! We'll reply to $email.")
    }
    name = ""
    email = ""
    message = ""
}
```

---

### ✅ Summary:

* ✔ Validates **name** for letters/spaces only
* ✔ Validates **email** format using regex
* ✔ Animates field borders and error messages
* ✔ Prevents form submission until all fields are valid

---

Great! Welcome to **Module 5: Navigation** — an essential part of any multi-screen Android app.

---

## ✅ Module 5: Navigation in Jetpack Compose

### 🎯 **Goal:** Build multi-screen apps using the Navigation component for Jetpack Compose.

---

## 🧱 Key Concepts

### 1. **Navigation in Compose:**

Jetpack Compose uses the `Navigation-Compose` library to handle screen transitions in a declarative way.

Add this dependency in your `build.gradle` (if not already present):

```kotlin
implementation("androidx.navigation:navigation-compose:2.7.7")
```

---

### 2. **Core Components**

| Component       | Purpose                                 |
| --------------- | --------------------------------------- |
| `NavController` | Controls navigation and back stack      |
| `NavHost`       | Hosts and switches between destinations |
| `composable()`  | Defines a screen                        |

---

## ✨ Simple Example: Two Screens (Home → Details)

### 🧩 Step 1: Define Screens

```kotlin
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Details : Screen("details/{username}") {
        fun createRoute(username: String) = "details/$username"
    }
}
```

---

### 🧩 Step 2: Setup NavHost in MainActivity

```kotlin
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController = navController, startDestination = Screen.Home.route) {
        composable(Screen.Home.route) {
            HomeScreen(
                onNavigate = { username ->
                    navController.navigate(Screen.Details.createRoute(username))
                }
            )
        }
        composable(Screen.Details.route) { backStackEntry ->
            val username = backStackEntry.arguments?.getString("username") ?: "Unknown"
            DetailsScreen(username)
        }
    }
}
```

---

### 🧩 Step 3: Home Screen

```kotlin
@Composable
fun HomeScreen(onNavigate: (String) -> Unit) {
    var input by remember { mutableStateOf("") }

    Column(
        Modifier
            .fillMaxSize()
            .wrapContentSize(Alignment.Center),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        OutlinedTextField(
            value = input,
            onValueChange = { input = it },
            label = { Text("Enter your name") }
        )

        Spacer(Modifier.height(16.dp))

        Button(onClick = {
            if (input.isNotBlank()) {
                onNavigate(input)
            }
        }) {
            Text("Go to Details")
        }
    }
}
```

---

### 🧩 Step 4: Details Screen

```kotlin
@Composable
fun DetailsScreen(username: String) {
    Box(
        Modifier
            .fillMaxSize()
            .wrapContentSize(Alignment.Center)
    ) {
        Text("Welcome, $username!", fontSize = 24.sp)
    }
}
```

---

### 🧩 Step 5: Set Up `MainActivity`

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val navController = rememberNavController()
            AppNavHost(navController)
        }
    }
}
```

---

## ✅ Summary:

* 🧭 Use `NavHost` to define your app’s routes
* 🧑 Pass arguments between screens using route parameters
* 🧨 Compose makes transitions simple and type-safe

---

# ✅ Navigation Advanced Features

## → Bottom Navigation

## → Backstack Handling

## → Animated Transitions

---

## 🧭 Part 1: Bottom Navigation Bar

### 🔧 Step 1: Define Destinations

```kotlin
sealed class BottomNavScreen(val route: String, val title: String, val icon: ImageVector) {
    object Home : BottomNavScreen("home", "Home", Icons.Default.Home)
    object Profile : BottomNavScreen("profile", "Profile", Icons.Default.Person)
    object Settings : BottomNavScreen("settings", "Settings", Icons.Default.Settings)
}
```

---

### 🧩 Step 2: Scaffold + Navigation + BottomBar

```kotlin
@Composable
fun MainScaffoldNav() {
    val navController = rememberNavController()
    val screens = listOf(
        BottomNavScreen.Home,
        BottomNavScreen.Profile,
        BottomNavScreen.Settings
    )

    Scaffold(
        bottomBar = {
            BottomNavigation {
                val currentRoute = navController.currentBackStackEntryAsState().value?.destination?.route
                screens.forEach { screen ->
                    BottomNavigationItem(
                        selected = currentRoute == screen.route,
                        onClick = {
                            if (currentRoute != screen.route) {
                                navController.navigate(screen.route) {
                                    popUpTo(navController.graph.startDestinationId) {
                                        saveState = true
                                    }
                                    launchSingleTop = true
                                    restoreState = true
                                }
                            }
                        },
                        icon = { Icon(screen.icon, contentDescription = screen.title) },
                        label = { Text(screen.title) }
                    )
                }
            }
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = BottomNavScreen.Home.route,
            modifier = Modifier.padding(innerPadding)
        ) {
            composable(BottomNavScreen.Home.route) { Text("🏠 Home Screen") }
            composable(BottomNavScreen.Profile.route) { Text("👤 Profile Screen") }
            composable(BottomNavScreen.Settings.route) { Text("⚙ Settings Screen") }
        }
    }
}
```

---

## 🔁 Part 2: Backstack Handling

Handled automatically by `NavController`. But:

* `launchSingleTop = true` avoids duplicate screen stacking.
* `restoreState = true` restores state when navigating back.
* `popUpTo()` controls how deep the backstack is cleared.

To control programmatic **back navigation**, use:

```kotlin
navController.popBackStack()
```

Or:

```kotlin
BackHandler {
    // Custom logic here
    navController.popBackStack()
}
```

---

## 🎞 Part 3: Animated Transitions (Optional but Cool)

### 🧩 Add Dependency

```kotlin
implementation("androidx.navigation:navigation-compose:2.7.7")
implementation("androidx.navigation:navigation-animation:2.7.7")
```

> ⚠️ For animations, you’ll likely use **Accompanist Navigation-Animation** (Google experimental library)

```kotlin
implementation("com.google.accompanist:accompanist-navigation-animation:0.34.0")
```

### 🧩 AnimatedNavHost

```kotlin
import com.google.accompanist.navigation.animation.AnimatedNavHost
import com.google.accompanist.navigation.animation.composable

@OptIn(ExperimentalAnimationApi::class)
@Composable
fun AnimatedNavGraph(navController: NavHostController) {
    AnimatedNavHost(
        navController = navController,
        startDestination = "screen1",
        enterTransition = { slideInHorizontally() },
        exitTransition = { slideOutHorizontally() }
    ) {
        composable("screen1") { ScreenOne(navController) }
        composable("screen2") { ScreenTwo(navController) }
    }
}
```

---

## ✅ Summary

| Feature           | Benefit                                 |
| ----------------- | --------------------------------------- |
| BottomNavigation  | Modern multi-tab UX                     |
| Backstack Control | Prevents stacking or incorrect nav flow |
| AnimatedNavHost   | Adds professional polish                |

---

# ✅ Module 6: Lists & Lazy Layouts

### 🎯 **Goal:** Use `LazyColumn`, `LazyRow`, and `LazyVerticalGrid` to build high-performance lists and grids.

---

## 📦 Core Components

### 1. **LazyColumn**

Efficient vertical list (only renders visible items)

```kotlin
LazyColumn {
    items(100) { index ->
        Text("Item #$index", modifier = Modifier.padding(16.dp))
    }
}
```

---

### 2. **LazyRow**

Horizontal scrolling layout

```kotlin
LazyRow {
    items(10) { index ->
        Card(modifier = Modifier.padding(8.dp)) {
            Text("Card #$index", modifier = Modifier.padding(16.dp))
        }
    }
}
```

---

### 3. **LazyVerticalGrid**

Requires:

```kotlin
implementation("androidx.compose.foundation:foundation:1.5.4")
```

Example:

```kotlin
LazyVerticalGrid(
    columns = GridCells.Fixed(2),
    modifier = Modifier.fillMaxSize(),
    contentPadding = PaddingValues(8.dp)
) {
    items(20) { index ->
        Card(
            modifier = Modifier
                .padding(8.dp)
                .fillMaxWidth()
        ) {
            Text("Grid item #$index", modifier = Modifier.padding(16.dp))
        }
    }
}
```

---

## ✨ Custom List Example: Contact Card

```kotlin
data class Contact(val name: String, val phone: String)

@Composable
fun ContactCard(contact: Contact) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(Modifier.padding(16.dp)) {
            Text(text = contact.name, fontSize = 20.sp, fontWeight = FontWeight.Bold)
            Text(text = contact.phone, color = Color.Gray)
        }
    }
}
```

### Using `LazyColumn`:

```kotlin
@Composable
fun ContactList(contacts: List<Contact>) {
    LazyColumn {
        items(contacts) { contact ->
            ContactCard(contact)
        }
    }
}
```

---

### 🧪 Practice Challenge

Build a **Product Catalog** with:

* A `LazyVerticalGrid`
* Each item shows an image, product name, and price
* Bonus: Add clickable interaction

Perfect — let’s build a **Product Catalog Grid** using `LazyVerticalGrid`.

---

## 🛍️ Product Catalog Grid

### 🎯 Features:

* Image, name, and price for each item
* 2-column grid layout
* Clickable cards with simple feedback

---

### 🧱 Step 1: Product Model

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val price: String,
    val imageRes: Int
)
```

You can store your images in `res/drawable/` (e.g., `product1.png`, `product2.png`, etc.)

---

### 🖼️ Step 2: Product Card

```kotlin
@Composable
fun ProductCard(product: Product, onClick: () -> Unit) {
    Card(
        modifier = Modifier
            .padding(8.dp)
            .fillMaxWidth()
            .clickable { onClick() },
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(
            modifier = Modifier.padding(8.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Image(
                painter = painterResource(id = product.imageRes),
                contentDescription = product.name,
                modifier = Modifier
                    .height(120.dp)
                    .fillMaxWidth()
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(product.name, fontWeight = FontWeight.Bold)
            Text(product.price, color = Color.Gray)
        }
    }
}
```

---

### 🧩 Step 3: Product Grid

```kotlin
@Composable
fun ProductGrid(products: List<Product>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        items(products) { product ->
            ProductCard(product = product, onClick = {
                println("Clicked: ${product.name}")
            })
        }
    }
}
```

---

### 🧪 Step 4: Preview with Sample Data

```kotlin
@Composable
@Preview(showBackground = true)
fun ProductGridPreview() {
    val sampleProducts = listOf(
        Product(1, "Sneakers", "$49.99", R.drawable.product1),
        Product(2, "Backpack", "$79.99", R.drawable.product2),
        Product(3, "Sunglasses", "$29.99", R.drawable.product3),
        Product(4, "Watch", "$199.99", R.drawable.product4)
    )

    ProductGrid(products = sampleProducts)
}
```

---

### 🔥 Optional Enhancements:

* Add a “favorite” toggle icon
* Use `AnimatedVisibility` for expanding descriptions
* Navigate to product details screen on click

---

Great — let’s add **pagination (infinite scroll)** to your product grid!

---

# 🔄 Infinite Scroll in Jetpack Compose with Lazy Layouts

### 🎯 Goal:

Load more items when the user scrolls near the bottom — just like endless product feeds.

---

## 🔧 Step-by-Step

### 🧱 1. Track the Scroll State

We’ll use `LazyGridState` or `LazyListState` to detect when the user reaches the bottom.

---

### 🧩 Updated ProductGrid with Infinite Scroll:

```kotlin
@Composable
fun PaginatedProductGrid(
    products: List<Product>,
    onLoadMore: () -> Unit
) {
    val gridState = rememberLazyGridState()

    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        state = gridState,
        contentPadding = PaddingValues(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        items(products) { product ->
            ProductCard(product = product, onClick = {
                println("Clicked: ${product.name}")
            })
        }
    }

    // Trigger onLoadMore when near the bottom
    LaunchedEffect(gridState) {
        snapshotFlow { gridState.layoutInfo }
            .collect { layoutInfo ->
                val totalItems = layoutInfo.totalItemsCount
                val lastVisible = layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
                if (lastVisible >= totalItems - 4) { // near bottom
                    onLoadMore()
                }
            }
    }
}
```

---

### 🧪 2. Usage with Dynamic List

```kotlin
@Composable
fun ProductGridWithPagination() {
    var productList by remember { mutableStateOf(generateProducts(0, 20)) }

    PaginatedProductGrid(
        products = productList,
        onLoadMore = {
            // Simulate network delay
            LaunchedEffect(Unit) {
                delay(1000) // Simulate loading time
                val next = generateProducts(productList.size, 10)
                productList = productList + next
            }
        }
    )
}
```

---

### 🧩 3. Generate Sample Products

```kotlin
fun generateProducts(start: Int, count: Int): List<Product> {
    return (start until start + count).map {
        Product(
            id = it,
            name = "Item $it",
            price = "$${(20..100).random()}.00",
            imageRes = R.drawable.product_placeholder // use a placeholder image
        )
    }
}
```

---

## ✅ Summary:

* `LazyGridState` monitors scroll position
* `LaunchedEffect + snapshotFlow` triggers `onLoadMore()` safely
* Simple and scalable pagination pattern

---

# 🔄 Add Loading Indicator to Lazy Grid Pagination

### 🎯 Goal:

Show a `CircularProgressIndicator` while loading new items during scroll.

---

## 🧩 Step 1: Add Loading State

Update your pagination container:

```kotlin
@Composable
fun ProductGridWithPagination() {
    var productList by remember { mutableStateOf(generateProducts(0, 20)) }
    var isLoading by remember { mutableStateOf(false) }

    PaginatedProductGrid(
        products = productList,
        isLoading = isLoading,
        onLoadMore = {
            if (!isLoading) {
                isLoading = true
                LaunchedEffect(Unit) {
                    delay(1500) // Simulate network delay
                    val next = generateProducts(productList.size, 10)
                    productList = productList + next
                    isLoading = false
                }
            }
        }
    )
}
```

---

## 🧩 Step 2: Modify `PaginatedProductGrid` to Show Loading Indicator

Update the grid function:

```kotlin
@Composable
fun PaginatedProductGrid(
    products: List<Product>,
    isLoading: Boolean,
    onLoadMore: () -> Unit
) {
    val gridState = rememberLazyGridState()

    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        state = gridState,
        contentPadding = PaddingValues(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        items(products) { product ->
            ProductCard(product = product, onClick = { /* ... */ })
        }

        // Add loading item
        if (isLoading) {
            item(span = { GridItemSpan(2) }) {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(24.dp),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
        }
    }

    // Detect near-end of list
    LaunchedEffect(gridState) {
        snapshotFlow { gridState.layoutInfo }
            .collect { layoutInfo ->
                val totalItems = layoutInfo.totalItemsCount
                val lastVisible = layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
                if (lastVisible >= totalItems - 4 && !isLoading) {
                    onLoadMore()
                }
            }
    }
}
```

---

## ✅ Result:

* Spinner appears in the grid while items are loading
* Pagination is smooth and user-friendly
* Prevents multiple concurrent loads

---

# 🎨 Module 7: Theming & Material 3

### 🎯 Goal: Customize colors, typography, shapes, and apply dark/light themes using Material Design 3 (M3).

---

## 🧱 1. Material 3 Setup (if not already using)

Make sure you’re using **Material3** theme in your `build.gradle`:

```kotlin
implementation("androidx.compose.material3:material3:1.2.1")
```

In your app theme, confirm this:

```kotlin
MaterialTheme(
    colorScheme = ...,     // Custom or dynamic
    typography = ...,      // Optional
    shapes = ...,          // Optional
    content = { /* App UI */ }
)
```

---

## 🌈 2. Light & Dark Themes

### 🔧 `Theme.kt` Setup

Usually in `ui/theme/Theme.kt`:

```kotlin
private val LightColors = lightColorScheme(
    primary = Color(0xFF006C4F),
    secondary = Color(0xFF4CAF50),
    background = Color(0xFFFDFDFD)
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFF80CBC4),
    secondary = Color(0xFFB2DFDB),
    background = Color(0xFF121212)
)

@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) DarkColors else LightColors

    MaterialTheme(
        colorScheme = colors,
        typography = Typography, // optional
        shapes = Shapes,         // optional
        content = content
    )
}
```

Apply in your `MainActivity`:

```kotlin
setContent {
    MyAppTheme {
        AppNavHost()
    }
}
```

---

## ✏️ 3. Typography

Define custom styles if needed:

```kotlin
val Typography = Typography(
    titleLarge = TextStyle(
        fontSize = 22.sp,
        fontWeight = FontWeight.Bold
    ),
    bodyMedium = TextStyle(
        fontSize = 16.sp
    )
)
```

Use it in your UI:

```kotlin
Text(
    text = "Welcome!",
    style = MaterialTheme.typography.titleLarge
)
```

---

## 🧩 4. Shapes

You can globally round your components:

```kotlin
val Shapes = Shapes(
    small = RoundedCornerShape(4.dp),
    medium = RoundedCornerShape(8.dp),
    large = RoundedCornerShape(16.dp)
)
```

---

## 🌗 5. Dynamic Color (Android 12+)

Enable Material You support (optional):

```kotlin
val dynamicScheme = if (darkTheme)
    dynamicDarkColorScheme(context)
else
    dynamicLightColorScheme(context)
```

---

## ✅ Summary

| Feature     | Purpose                               |
| ----------- | ------------------------------------- |
| ColorScheme | Controls surface, primary, background |
| Typography  | Title/body fonts, sizes, weights      |
| Shapes      | Button corners, card roundness        |
| Theme.kt    | Centralizes styling logic             |

---

# 🎞️ Module 8: Animations in Jetpack Compose

### 🎯 Goal: Use built-in Compose animation APIs to animate visibility, state changes, transitions, and gestures.

---

## 🔧 1. Simple State-Based Animation

### animate*AsState

Smoothly animate between values when state changes (like colors, sizes, positions):

```kotlin
val isActive by remember { mutableStateOf(false) }
val bgColor by animateColorAsState(if (isActive) Color.Green else Color.Red)

Box(
    modifier = Modifier
        .size(100.dp)
        .background(bgColor)
        .clickable { isActive = !isActive }
)
```

---

## 🧍 2. AnimatedVisibility

Fade, slide, or expand content based on visibility state:

```kotlin
var visible by remember { mutableStateOf(true) }

Column {
    Button(onClick = { visible = !visible }) {
        Text(if (visible) "Hide" else "Show")
    }

    AnimatedVisibility(
        visible = visible,
        enter = fadeIn() + expandVertically(),
        exit = shrinkVertically() + fadeOut()
    ) {
        Text("Animated content here", modifier = Modifier.padding(16.dp))
    }
}
```

---

## 🔁 3. updateTransition

Animate multiple properties in sync:

```kotlin
val selected by remember { mutableStateOf(false) }
val transition = updateTransition(targetState = selected, label = "Box Transition")

val size by transition.animateDp(label = "Size") {
    if (it) 150.dp else 80.dp
}

val color by transition.animateColor(label = "Color") {
    if (it) Color.Blue else Color.Gray
}

Box(
    modifier = Modifier
        .size(size)
        .background(color)
        .clickable { selected = !selected }
)
```

---

## 🌀 4. AnimatedContent (state crossfades)

```kotlin
var count by remember { mutableStateOf(0) }

Column(horizontalAlignment = Alignment.CenterHorizontally) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            slideInVertically() + fadeIn() with slideOutVertically() + fadeOut()
        }
    ) { targetCount ->
        Text("Count: $targetCount", fontSize = 24.sp)
    }

    Button(onClick = { count++ }) {
        Text("Increment")
    }
}
```

---

## 🧪 Challenge

Animate a **Product Card** so that when clicked:

* It **expands** vertically
* Shows more details (e.g., description, buttons)
* Use `AnimatedVisibility` or `updateTransition`

Great! Let's build a fully functional **Expandable Product Card** that smoothly reveals more product details on click.

---

# 🛍️ Expandable Product Card with Animation

### 🎯 Features:

* Card expands/collapses on click
* Shows additional info (e.g., description, rating, buttons)
* Smooth height + content animation using `AnimatedVisibility` and `animateDpAsState`

---

### ✅ Step 1: Sample Product Data

```kotlin
data class Product(
    val name: String,
    val price: String,
    val description: String,
    val imageRes: Int
)
```

---

### ✅ Step 2: Expandable Card Implementation

```kotlin
@Composable
fun ExpandableProductCard(product: Product) {
    var expanded by remember { mutableStateOf(false) }

    val cardHeight by animateDpAsState(
        targetValue = if (expanded) 260.dp else 180.dp,
        label = "Card Height"
    )

    Card(
        modifier = Modifier
            .padding(8.dp)
            .fillMaxWidth()
            .height(cardHeight)
            .clickable { expanded = !expanded },
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(
            modifier = Modifier
                .padding(12.dp)
                .fillMaxSize()
        ) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Image(
                    painter = painterResource(id = product.imageRes),
                    contentDescription = product.name,
                    contentScale = ContentScale.Crop,
                    modifier = Modifier
                        .size(64.dp)
                        .clip(RoundedCornerShape(8.dp))
                )
                Spacer(Modifier.width(12.dp))
                Column {
                    Text(product.name, fontWeight = FontWeight.Bold)
                    Text(product.price, color = Color.Gray)
                }
            }

            Spacer(Modifier.height(12.dp))

            AnimatedVisibility(visible = expanded) {
                Column {
                    Text(
                        text = product.description,
                        style = MaterialTheme.typography.bodyMedium,
                        maxLines = 4,
                        overflow = TextOverflow.Ellipsis
                    )
                    Spacer(Modifier.height(12.dp))
                    Button(onClick = { /* Add to cart logic */ }) {
                        Text("Add to Cart")
                    }
                }
            }
        }
    }
}
```

---

### ✅ Step 3: Preview With Sample Product

```kotlin
@Preview(showBackground = true)
@Composable
fun ExpandableProductCardPreview() {
    val sampleProduct = Product(
        name = "Wireless Headphones",
        price = "$89.99",
        description = "High-quality noise-cancelling headphones with up to 30 hours of battery life, perfect for travel and work.",
        imageRes = R.drawable.product_placeholder // Replace with your drawable
    )

    ExpandableProductCard(sampleProduct)
}
```

---

### 💡 Enhancements (optional):

* Add an expand/collapse **arrow icon**
* Animate icon rotation with `animateFloatAsState`
* Track expanded state in a **ViewModel** for multiple cards

---

Excellent! You're now ready for **Module 9: Architecture & ViewModel** — where we structure apps to be scalable, testable, and maintainable.

---

# 🏗️ Module 9: Architecture & ViewModel

### 🎯 Goal:

Apply **MVVM (Model-View-ViewModel)** architecture in a Jetpack Compose app using **ViewModel**, **StateFlow**, and best practices.

---

## 🧱 1. What is MVVM?

| Layer     | Role                                             |
| --------- | ------------------------------------------------ |
| Model     | Business/data logic (DB, API, Repository)        |
| ViewModel | Holds UI state & logic, exposes observable state |
| View (UI) | Composables observe state and react to changes   |

---

## 🔧 2. Add Dependencies

In `build.gradle`:

```kotlin
implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
```

---

## 🧩 3. Sample Scenario: To-Do App

We’ll use a simple ViewModel to manage a list of tasks.

---

### 📄 Model

```kotlin
data class Task(val id: Int, val text: String, val isDone: Boolean = false)
```

---

### 🔮 ViewModel

```kotlin
class TaskViewModel : ViewModel() {
    private val _tasks = MutableStateFlow<List<Task>>(emptyList())
    val tasks: StateFlow<List<Task>> = _tasks

    private var nextId = 0

    fun addTask(text: String) {
        if (text.isBlank()) return
        _tasks.value = _tasks.value + Task(nextId++, text)
    }

    fun toggleTask(id: Int) {
        _tasks.value = _tasks.value.map {
            if (it.id == id) it.copy(isDone = !it.isDone) else it
        }
    }

    fun removeTask(id: Int) {
        _tasks.value = _tasks.value.filter { it.id != id }
    }
}
```

---

### 👁️ View (Compose)

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskViewModel = viewModel()) {
    val tasks by viewModel.tasks.collectAsState()

    var newTask by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        Row {
            OutlinedTextField(
                value = newTask,
                onValueChange = { newTask = it },
                label = { Text("New Task") },
                modifier = Modifier.weight(1f)
            )
            Spacer(Modifier.width(8.dp))
            Button(onClick = {
                viewModel.addTask(newTask)
                newTask = ""
            }) {
                Text("Add")
            }
        }

        Spacer(Modifier.height(16.dp))

        LazyColumn {
            items(tasks) { task ->
                Row(
                    verticalAlignment = Alignment.CenterVertically,
                    modifier = Modifier
                        .fillMaxWidth()
                        .clickable { viewModel.toggleTask(task.id) }
                        .padding(8.dp)
                ) {
                    Checkbox(
                        checked = task.isDone,
                        onCheckedChange = { viewModel.toggleTask(task.id) }
                    )
                    Text(
                        text = task.text,
                        textDecoration = if (task.isDone) TextDecoration.LineThrough else null,
                        modifier = Modifier.weight(1f)
                    )
                    IconButton(onClick = { viewModel.removeTask(task.id) }) {
                        Icon(Icons.Default.Delete, contentDescription = "Delete")
                    }
                }
            }
        }
    }
}
```

---

## ✅ Summary

| Component      | Purpose                            |
| -------------- | ---------------------------------- |
| ViewModel      | Stores and exposes UI state        |
| StateFlow      | Reactive state container           |
| collectAsState | Observe ViewModel state in Compose |

---

# 🗃️ Module 10: Data & Storage in Jetpack Compose

### 🎯 Goal:

Persist and manage data locally using **Room Database** and **DataStore**.

---

## 🧱 Two Key Tools

| Tool          | Use Case                                                      |
| ------------- | ------------------------------------------------------------- |
| **Room**      | Store structured data (SQLite)                                |
| **DataStore** | Store key-value settings (like SharedPreferences replacement) |

---

## 🔄 Part 1: Room + Compose + ViewModel

### 🧩 1. Add Room Dependencies

```kotlin
implementation("androidx.room:room-runtime:2.6.1")
kapt("androidx.room:room-compiler:2.6.1")
implementation("androidx.room:room-ktx:2.6.1")
```

Also enable `kapt` in `build.gradle`:

```kotlin
apply plugin: 'kotlin-kapt'
```

---

### 📄 2. Define Entity

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val text: String,
    val isDone: Boolean = false
)
```

---

### 🧠 3. DAO Interface

```kotlin
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY id DESC")
    fun getAll(): Flow<List<TaskEntity>>

    @Insert
    suspend fun insert(task: TaskEntity)

    @Delete
    suspend fun delete(task: TaskEntity)

    @Update
    suspend fun update(task: TaskEntity)
}
```

---

### 🏛 4. Database Class

```kotlin
@Database(entities = [TaskEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
}
```

---

### 🧩 5. Repository + ViewModel

```kotlin
class TaskRepository(private val dao: TaskDao) {
    val tasks = dao.getAll()

    suspend fun add(text: String) = dao.insert(TaskEntity(text = text))
    suspend fun toggle(task: TaskEntity) =
        dao.update(task.copy(isDone = !task.isDone))
    suspend fun remove(task: TaskEntity) = dao.delete(task)
}
```

```kotlin
class TaskDbViewModel(private val repo: TaskRepository) : ViewModel() {
    val tasks = repo.tasks.stateIn(viewModelScope, SharingStarted.WhileSubscribed(), emptyList())

    fun add(text: String) {
        viewModelScope.launch { repo.add(text) }
    }

    fun toggle(task: TaskEntity) {
        viewModelScope.launch { repo.toggle(task) }
    }

    fun remove(task: TaskEntity) {
        viewModelScope.launch { repo.remove(task) }
    }
}
```

---

### 🧪 Integration with Compose

Same UI pattern as before, just update your list from `TaskEntity` and call `viewModel.add(...)`.

---

## ⚙️ Part 2: Preferences with DataStore

### 🧩 Add Dependency

```kotlin
implementation("androidx.datastore:datastore-preferences:1.0.0")
```

---

### ✏️ Preferences Example

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore("settings")

object PreferenceKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
}

suspend fun setDarkMode(context: Context, enabled: Boolean) {
    context.dataStore.edit { it[PreferenceKeys.DARK_MODE] = enabled }
}

fun getDarkModeFlow(context: Context): Flow<Boolean> =
    context.dataStore.data.map { prefs ->
        prefs[PreferenceKeys.DARK_MODE] ?: false
    }
```

---

## ✅ Summary

| Tool      | Use For                                     |
| --------- | ------------------------------------------- |
| Room      | Persistent, structured data (e.g., tasks)   |
| DataStore | Small settings or toggles (e.g., dark mode) |

---

Excellent choice! Let’s enhance your app by adding a **dark mode toggle** using **DataStore**, which will persist the user's theme preference across app restarts.

---

# 🌗 Add Dark Mode Toggle with DataStore + Compose

### 🎯 Goal:

Allow users to switch between light and dark mode, and **remember their choice** using `DataStore`.

---

## 🔧 Step 1: Add DataStore Dependency

In `build.gradle`:

```kotlin
implementation("androidx.datastore:datastore-preferences:1.0.0")
```

---

## 🧱 Step 2: Create DataStore Helper

```kotlin
object SettingsDataStore {
    private val Context.dataStore by preferencesDataStore(name = "settings")

    private val DARK_MODE = booleanPreferencesKey("dark_mode")

    fun getDarkModeFlow(context: Context): Flow<Boolean> =
        context.dataStore.data.map { it[DARK_MODE] ?: false }

    suspend fun setDarkMode(context: Context, enabled: Boolean) {
        context.dataStore.edit { it[DARK_MODE] = enabled }
    }
}
```

---

## 🧠 Step 3: ThemeViewModel

```kotlin
class ThemeViewModel(context: Context) : ViewModel() {
    private val _darkMode = MutableStateFlow(false)
    val darkMode: StateFlow<Boolean> = _darkMode

    private val appContext = context.applicationContext

    init {
        viewModelScope.launch {
            SettingsDataStore.getDarkModeFlow(appContext).collect {
                _darkMode.value = it
            }
        }
    }

    fun toggleTheme() {
        viewModelScope.launch {
            val newMode = !_darkMode.value
            SettingsDataStore.setDarkMode(appContext, newMode)
        }
    }
}
```

---

## 🎨 Step 4: Use Theme in App

Wrap your app content with a theme that listens to `darkMode`.

```kotlin
@Composable
fun MyApp(viewModel: ThemeViewModel, content: @Composable () -> Unit) {
    val darkMode by viewModel.darkMode.collectAsState()

    MaterialTheme(
        colorScheme = if (darkMode) darkColorScheme() else lightColorScheme(),
        typography = Typography,
        content = content
    )
}
```

In your `MainActivity`:

```kotlin
val themeViewModel: ThemeViewModel = viewModel(factory = YourViewModelFactory(context))

MyApp(viewModel = themeViewModel) {
    TaskDbScreen() // or your root composable
}
```

---

## 🖼 Step 5: Toggle Switch in UI

```kotlin
@Composable
fun ThemeToggle(viewModel: ThemeViewModel) {
    val darkMode by viewModel.darkMode.collectAsState()

    Row(verticalAlignment = Alignment.CenterVertically) {
        Text("Dark Mode")
        Switch(
            checked = darkMode,
            onCheckedChange = { viewModel.toggleTheme() }
        )
    }
}
```

Place `ThemeToggle()` at the top or in settings screen.

---

## ✅ Summary

| Component      | Role                          |
| -------------- | ----------------------------- |
| DataStore      | Persists dark mode toggle     |
| ThemeViewModel | Holds current theme state     |
| MaterialTheme  | Applies UI theme reactively   |
| Switch         | Lets user control the setting |