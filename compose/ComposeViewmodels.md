# Viewmodels 

The `ViewModel` class is a business logic or screen level state holder. It exposes state to the UI and encapsulates related business logic. Its principal advantage is that it caches state and persists it through configuration changes. This means that your UI doesn’t have to fetch data again when navigating between activities, or following configuration changes, such as when rotating the screen.

The key benefits of the ViewModel class are essentially two:
- It allows you to persist UI state.
- It provides access to business logic.

## Persistence

ViewModel allows persistence through both the state that a ViewModel holds, and the operations that a ViewModel triggers. This caching means that you don’t have to fetch data again through common configuration changes, such as a screen rotation.

## Scope

When you instantiate a ViewModel, you pass it an object that implements the `ViewModelStoreOwner` interface. This may be a Navigation destination, Navigation graph, activity, fragment, or any other type that implements the interface. Your ViewModel is then scoped to the Lifecycle of the ViewModelStoreOwner. It remains in memory until its ViewModelStoreOwner goes away permanently.

A range of classes are either direct or indirect subclasses of the ViewModelStoreOwner interface. The direct subclasses are `ComponentActivity`, `Fragment`, and `NavBackStackEntry`.

When the fragment or activity to which the ViewModel is scoped is destroyed, asynchronous work continues in the ViewModel that is scoped to it. This is the key to persistence.

## Jetpack Compose

With Jetpack Compose, ViewModel is the primary means of exposing screen UI state to your composables. In a hybrid app, activities and fragments simply host your composable functions. This is a shift from past approaches, where it wasn't that simple and intuitive to create reusable pieces of UI with activities and fragments, which caused them to be much more active as UI controllers.

The most important thing to keep in mind when using ViewModel with Compose is that you cannot scope a ViewModel to a composable. This is because a composable is not a ViewModelStoreOwner. Two instances of the same composable in the Composition, or two different composables accessing the same ViewModel type under the same ViewModelStoreOwner would receive the same instance of the ViewModel, which often is not the expected behavior.

> To get the benefits of ViewModel in Compose, host each screen in a Fragment or Activity, or use Compose Navigation and use ViewModels in composable functions as close as possible to the Navigation destination. That is because you can scope a ViewModel to Navigation destinations, Navigation graphs, Activities, and Fragments.

# Implement ViewModel

The following is an example implementation of a ViewModel for a screen that allows the user to roll dice.
Important: In this example, the responsibility of acquiring and holding the list of users sits with the ViewModel, not an Activity or Fragment directly.


```kotlin
data class DiceUiState(
    val firstDieValue: Int? = null,
    val secondDieValue: Int? = null,
    val numberOfRolls: Int = 0,
)
```

```kotlin
class DiceRollViewModel : ViewModel() {

    // Expose screen UI state
    private val _uiState = MutableStateFlow(DiceUiState())
    val uiState: StateFlow<DiceUiState> = _uiState.asStateFlow()

    // Handle business logic
    fun rollDice() {
        _uiState.update { currentState ->
            currentState.copy(
                firstDieValue = Random.nextInt(from = 1, until = 7),
                secondDieValue = Random.nextInt(from = 1, until = 7),
                numberOfRolls = currentState.numberOfRolls + 1,
            )
        }
    }
}
```

You can then access the ViewModel from an activity as follows:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel

// Use the 'viewModel()' function from the lifecycle-viewmodel-compose artifact
@Composable
fun DiceRollScreen(
    viewModel: DiceRollViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // Update UI elements
}
```

> Caution: A ViewModel usually shouldn't reference a view, Lifecycle, or any class that may hold a reference to the activity context. Because the ViewModel lifecycle is larger than the UI's, holding a lifecycle-related API in the ViewModel could cause memory leaks.

> Note: To import ViewModel into your Android project, see the instructions for declaring dependencies in the Lifecycle release notes.

# ViewModel

The ViewModel component holds and exposes the state the UI consumes. The UI state is application data transformed by ViewModel. ViewModel lets your app follow the architecture principle of driving the UI from the model.

ViewModel stores the app-related data that isn't destroyed when the activity is destroyed and recreated by the Android framework. Unlike the activity instance, ViewModel objects are not destroyed. The app automatically retains ViewModel objects during configuration changes so that the data they hold is immediately available after the recomposition.

To implement ViewModel in your app, extend the ViewModel class, which comes from the architecture components library and stores app data within that class.

## UI State

The UI is what the user sees, and the UI state is what the app says they should see. The UI is the visual representation of the UI state. Any changes to the UI state immediately are reflected in the UI.

```
+-------------+     +----------+         /''''''\
| UI Elements |  +  | UI State |    =   (   UI   )
+-------------+     +----------+         \______/
```
UI is a result of binding UI elements on the screen with the UI state.


```kotlin
// Example of UI state definition, do not copy over
data class NewsItemUiState(
    val title: String,
    val body: String,
    val bookmarked: Boolean = false,
    ...
)
```

## Immutability

The UI state definition in the example above is immutable. Immutable objects provide guarantees that multiple sources do not alter the state of the app at an instant in time. This protection frees the UI to focus on a single role: reading state and updating UI elements accordingly. Therefore, you should never modify the UI state in the UI directly, unless the UI itself is the sole source of its data. Violating this principle results in multiple sources of truth for the same piece of information, leading to data inconsistencies and subtle bugs.

## StateFlow

`StateFlow` is a data holder observable flow that emits the current and new state updates. Its value property reflects the current state value. To update state and send it to the flow, assign a new value to the value property of the MutableStateFlow class.

In Android, StateFlow works well with classes that must maintain an observable immutable state.

__A StateFlow can be exposed from the GameUiState so that the composables can listen for UI state updates and make the screen state survive configuration changes.__

In the GameViewModel class, add the following _uiState property.

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow

// Game UI state
private val _uiState = MutableStateFlow(GameUiState())
```

## Backing property

A backing property lets you return something from a getter other than the exact object.

For var property, the Kotlin framework generates getters and setters.

For getter and setter methods, you can override one or both of these methods and provide your own custom behavior. To implement a backing property, you override the getter method to return a read-only version of your data. The following example shows a backing property:

```kotlin
// Declare private mutable variable that can only be modified
// within the class it is declared.
private var _count = 0 

// Declare another public immutable field and override its getter method. 
// Return the private property's value in the getter method.
// When count is accessed, the get() function is called and
// the value of _count is returned. 
val count: Int
    get() = _count
```

As another example, say that you want the app data to be private to the ViewModel:

Inside the ViewModel class:
- The property _count is private and mutable. Hence, it is only accessible and editable within the ViewModel class.

Outside the ViewModel class:

- The default visibility modifier in Kotlin is public, so count is public and accessible from other classes like UI controllers. A val type cannot have a setter. It is immutable and read-only so you can only override the get() method. When an outside class accesses this property, it returns the value of _count and its value can't be modified. This backing property protects the app data inside the ViewModel from unwanted and unsafe changes by external classes, but it lets external callers safely access its value.

In the GameViewModel.kt file, add a backing property to uiState named _uiState. Name the property uiState and is of the type StateFlow<GameUiState>.

Now _uiState is accessible and editable only within the GameViewModel. The UI can read its value using the read-only property, uiState. You can fix the initialization error in the next step.

```kotlin
import kotlinx.coroutines.flow.StateFlow

// Game UI state

// Backing property to avoid state updates from other classes
private val _uiState = MutableStateFlow(GameUiState())
val uiState: StateFlow<GameUiState> 
```

Set uiState to `_uiState.asStateFlow()`.

The `asStateFlow()` makes this mutable state flow a read-only state flow.

```kotlin
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

// Game UI state
private val _uiState = MutableStateFlow(GameUiState())
val uiState: StateFlow<GameUiState> = _uiState.asStateFlow()
```

```kotlin
fun resetGame() {
   usedWords.clear()
   _uiState.value = GameUiState(currentScrambledWord = pickRandomWordAndShuffle())
}
```

## Unidirectional data flow

A unidirectional data flow (UDF) is a design pattern in which state flows down and events flow up. By following unidirectional data flow, you can decouple composables that display state in the UI from the parts of your app that store and change state.

The UI update loop for an app using unidirectional data flow looks like the following:
- Event: Part of the UI generates an event and passes it upward—such as a button click passed to the ViewModel to handle—or an event that is passed from other layers of your app, such as an indication that the user session has expired.
- Update state: An event handler might change the state.
- Display state: The state holder passes down the state, and the UI displays it.

```
   +-----------------+
   |   Data Layer    |
   +-----------------+
        : Application
        : data
        v
+-----------------------+
|       UI Layer        |
|  +-----------------+  |
|  |    ViewModel    |  |
|  +-----------------+  |
|     ^        |        |
|     |        v        |
| events  +---------+   |
|     |   | UI      |   |
|     |   | state   |   |
|     |   +---------+   |
|     |       |         |
|     |       v         |
|  +-----------------+  |
|  |   UI elements   |  |
|  +-----------------+  |
+-----------------------+

```
The use of the UDF pattern for app architecture has the following implications:
- The ViewModel holds and exposes the state the UI consumes.
- The UI state is application data transformed by the ViewModel.
- The UI notifies the ViewModel of user events.
- The ViewModel handles the user actions and updates the state.
- The updated state is fed back to the UI to render.
- This process repeats for any event that causes a mutation of state.

## Pass the data

Pass the ViewModel instance to the UI–that is, from the GameViewModel to the GameScreen() in the GameScreen.kt file. In the GameScreen(), use the ViewModel instance to access the uiState using collectAsState().

The collectAsState() function collects values from this StateFlow and represents its latest value via State. The StateFlow.value is used as an initial value. Every time there would be a new value posted into the StateFlow, the returned State updates, causing recomposition of every State.value usage.

1. In the GameScreen function, pass a second argument of the type GameViewModel with a default value of viewModel().

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun GameScreen(
   gameViewModel: GameViewModel = viewModel()
) {
   // ...
}
```

2. In the GameScreen() function, add a new variable called gameUiState. Use the by delegate and call collectAsState() on uiState.

This approach ensures that whenever there is a change in the uiState value, recomposition occurs for the composables using the gameUiState value.

```kotlin
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue

@Composable
fun GameScreen(
   // ...
) {
   val gameUiState by gameViewModel.uiState.collectAsState()
   // ...
}
```

3. Pass the gameUiState.currentScrambledWord to the GameLayout() composable. You add the argument in a later step, so ignore the error for now.

```kotlin
GameLayout(
   currentScrambledWord = gameUiState.currentScrambledWord,
   modifier = Modifier
       .fillMaxWidth()
       .wrapContentHeight()
       .padding(mediumPadding)
)
```

4. Add currentScrambledWord as another parameter to the GameLayout() composable function.

```kotlin
@Composable
fun GameLayout(
   currentScrambledWord: String,
   modifier: Modifier = Modifier
) {
}
```

## Display the guess word

In the GameLayout() composable, updating the user's guess word is one of event callbacks that flows up from GameScreen to the ViewModel. The data gameViewModel.userGuess will flow down from the ViewModel to the GameScreen.

```
+------------------------------------------------------+
|                   GameViewModel                      |
|             +-------------------------+              |
|             |     GameUiState         |              |
|             +-------------------------+              |
+---------------------------------------^--------------+      
    :  gameUiState.currentScrambledWord | onKeyboardDone()
    :  gameViewModel.userGuess          | onUserGuessChanged()
    v                                   |
+------------------------------------------------------+
|             GameScreen.kt - GameLayout()             |
+------------------------------------------------------+
                   
```

1. In the GameScreen.kt file, in the GameLayout() composable, set onValueChange to onUserGuessChanged and onKeyboardDone() to onDone keyboard action. You fix the errors in the next step.

```kotlin
OutlinedTextField(
   value = "",
   singleLine = true,
   modifier = Modifier.fillMaxWidth(),
   onValueChange = onUserGuessChanged,
   label = { Text(stringResource(R.string.enter_your_word)) },
   isError = false,
   keyboardOptions = KeyboardOptions.Default.copy(
       imeAction = ImeAction.Done
   ),
   keyboardActions = KeyboardActions(
       onDone = { onKeyboardDone() }
   ),
```

2. In the GameLayout() composable function, add two more arguments: the onUserGuessChanged lambda takes a String argument and returns nothing, and the onKeyboardDone takes nothing and returns nothing.

```kotlin
@Composable
fun GameLayout(
   onUserGuessChanged: (String) -> Unit,
   onKeyboardDone: () -> Unit,
   currentScrambledWord: String,
   modifier: Modifier = Modifier,
   ) {
}
```

3. In the GameLayout() function call, add lambda arguments for onUserGuessChanged and onKeyboardDone.

```kotlin
GameLayout(
   onUserGuessChanged = { gameViewModel.updateUserGuess(it) },
   onKeyboardDone = { },
   currentScrambledWord = gameUiState.currentScrambledWord,
)
```

You define updateUserGuess method in GameViewModel shortly.

4. In the GameViewModel.kt file, add a method called updateUserGuess() that takes a String argument, the user's guess word. Inside the function, update the userGuess with the passed in guessedWord.

```kotlin
  fun updateUserGuess(guessedWord: String){
     userGuess = guessedWord
  }
```

You add userGuess in the ViewModel next.

5. In the GameViewModel.kt file, add a var property called userGuess. Use mutableStateOf() so that Compose observes this value and sets the initial value to "".

```kotlin
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue


var userGuess by mutableStateOf("")
   private set
```

6. In the GameScreen.kt file, inside GameLayout(), add another String parameter for userGuess. Set the value parameter of the OutlinedTextField to userGuess.

```kotlin
fun GameLayout(
   currentScrambledWord: String,
   userGuess: String,
   onUserGuessChanged: (String) -> Unit,
   onKeyboardDone: () -> Unit,
   modifier: Modifier = Modifier
) {
   Column(
       verticalArrangement = Arrangement.spacedBy(24.dp)
   ) {
       //...
       OutlinedTextField(
           value = userGuess,
           //..
       )
   }
}
```

7. In the GameScreen function, update the GameLayout() function call to include userGuess parameter.

```
GameLayout(
   currentScrambledWord = gameUiState.currentScrambledWord,
   userGuess = gameViewModel.userGuess,
   onUserGuessChanged = { gameViewModel.updateUserGuess(it) },
   onKeyboardDone = { },
   //...
)
```

Functions to update the state:

```kotlin
private fun updateGameState(updatedScore: Int) {
   _uiState.update { currentState ->
       currentState.copy(
           isGuessedWordWrong = false,
           currentScrambledWord = pickRandomWordAndShuffle(),
           score = updatedScore
       )
   }
}
```

```kotlin
fun checkUserGuess() {
   if (userGuess.equals(currentWord, ignoreCase = true)) {
       // User's guess is correct, increase the score
       // and call updateGameState() to prepare the game for next round
       val updatedScore = _uiState.value.score.plus(SCORE_INCREASE)
       updateGameState(updatedScore)
   } else {
       // User's guess is wrong, show an error
       _uiState.update { currentState ->
           currentState.copy(isGuessedWordWrong = true)
       }
   }
   // Reset user guess
   updateUserGuess("")
}
```

## Sample of an alert dialog:

```kotlin
@Composable
private fun FinalScoreDialog(
   onPlayAgain: () -> Unit,
   modifier: Modifier = Modifier
) {
   val activity = (LocalContext.current as Activity)

   AlertDialog(
       onDismissRequest = {
           // Dismiss the dialog when the user clicks outside the dialog or on the back
           // button. If you want to disable that functionality, simply use an empty
           // onDismissRequest.
       },
       title = { Text(stringResource(R.string.congratulations)) },
       text = { Text(stringResource(R.string.you_scored, 0)) },
       modifier = modifier,
       dismissButton = {
           TextButton(
               onClick = {
                   activity.finish()
               }
           ) {
               Text(text = stringResource(R.string.exit))
           }
       },
       confirmButton = {
           TextButton(
               onClick = {
                   onPlayAgain()
               }
           ) {
               Text(text = stringResource(R.string.play_again))
           }
       }
   )
}
```