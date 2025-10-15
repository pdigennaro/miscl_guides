# Compose states

Key Term: Composition: a description of the UI built by Jetpack Compose when it executes composables.

Initial composition: creation of a Composition by running composables the first time.

Recomposition: re-running composables to update the Composition when data changes.

## State in composables

There are three ways to declare a MutableState object in a composable:
- `val mutableState = remember { mutableStateOf(default) }`
- `var value by remember { mutableStateOf(default) }`
- `val (value, setValue) = remember { mutableStateOf(default) }`

You can use the remembered value as a parameter for other composables or even as logic in statements to change which composables are displayed. For example, if you don't want to display the greeting if the name is empty, use the state in an if statement:

```kotlin
@Composable
fun HelloContent() {
    Column(modifier = Modifier.padding(16.dp)) {
        var name by remember { mutableStateOf("") }
        if (name.isNotEmpty()) {
            Text(
                text = "Hello, $name!",
                modifier = Modifier.padding(bottom = 8.dp),
                style = MaterialTheme.typography.bodyMedium
            )
        }
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Name") }
        )
    }
}
```

While remember helps you retain state across recompositions, the state is not retained across configuration changes. For this, you must use rememberSaveable. rememberSaveable automatically saves any value that can be saved in a Bundle. For other values, you can pass in a custom saver object.
Caution: Using mutable objects such as ArrayList<T> or mutableListOf() as state in Compose causes your users to see incorrect or stale data in your app. Mutable objects that are not observable, such as ArrayList or a mutable data class, are not observable by Compose and don't trigger a recomposition when they change. Instead of using non-observable mutable objects, the recommendation is to use an observable data holder such as `State<List<T>>` and the immutable `listOf()`.

## Stateful versus stateless

A composable that uses remember to store an object creates internal state, making the composable stateful. HelloContent is an example of a stateful composable because it holds and modifies its name state internally. This can be useful in situations where a caller doesn't need to control the state and can use it without having to manage the state themselves. However, composables with internal state tend to be less reusable and harder to test.

A stateless composable is a composable that doesn't hold any state. An easy way to achieve stateless is by using state hoisting.

As you develop reusable composables, you often want to expose both a stateful and a stateless version of the same composable. The stateful version is convenient for callers that don't care about the state, and the stateless version is necessary for callers that need to control or hoist the state.

## State hoisting

State hoisting in Compose is a pattern of moving state to a composable's caller to make a composable stateless. The general pattern for state hoisting in Jetpack Compose is to replace the state variable with two parameters:
- value: T: the current value to display
- onValueChange: (T) -> Unit: an event that requests the value to change, where T is the proposed new value

However, you are not limited to onValueChange. If more specific events are appropriate for the composable, you should define them using lambdas.

State that is hoisted this way has some important properties:
- Single source of truth: By moving state instead of duplicating it, we're ensuring there's only one source of truth. This helps avoid bugs.
- Encapsulated: Only stateful composables can modify their state. It's completely internal.
- Shareable: Hoisted state can be shared with multiple composables. If you wanted to read name in a different composable, hoisting would allow you to do that.
- Interceptable: callers to the stateless composables can decide to ignore or modify events before changing the state.
- Decoupled: the state for the stateless composables may be stored anywhere. For example, it's now possible to move name into a ViewModel.

In the example case, you extract the name and the onValueChange out of HelloContent and move them up the tree to a HelloScreen composable that calls HelloContent.

```kotlin
@Composable
fun HelloScreen() {
    var name by rememberSaveable { mutableStateOf("") }

    HelloContent(name = name, onNameChange = { name = it })
}
```

```kotlin
@Composable
fun HelloContent(name: String, onNameChange: (String) -> Unit) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(
            text = "Hello, $name",
            modifier = Modifier.padding(bottom = 8.dp),
            style = MaterialTheme.typography.bodyMedium
        )
        OutlinedTextField(value = name, onValueChange = onNameChange, label = { Text("Name") })
    }
}
```

By hoisting the state out of HelloContent, it's easier to reason about the composable, reuse it in different situations, and test. HelloContent is decoupled from how its state is stored. Decoupling means that if you modify or replace HelloScreen, you don't have to change how HelloContent is implemented.

```
┌────────────────┐
│  HelloScreen   │
└────────────────┘
        │
   state│▼
        │
┌────────────────┐
│  HelloContent  │
└────────────────┘
        ▲
   event││
```


The pattern where the state goes down, and events go up is called a unidirectional data flow. In this case, the state goes down from HelloScreen to HelloContent and events go up from HelloContent to HelloScreen. By following unidirectional data flow, you can decouple composables that display state in the UI from the parts of your app that store and change state.

> **Key Point:** When hoisting state, there are three rules to help you figure out where state should go:  
> 1. State should be hoisted to at least the lowest common parent of all composables that use the state (read).  
> 2. State should be hoisted to at least the highest level it may be changed (write).  
> 3. If two states change in response to the same events they should be hoisted together.  
> You can hoist state higher than these rules require, but underhoisting state makes it difficult or impossible to follow unidirectional data flow.