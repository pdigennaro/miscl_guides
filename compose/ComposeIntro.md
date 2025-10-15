# Lifecycle overview

A ``Composition`` describes the UI of your app and is produced by running composables. A Composition is a tree-structure of the composables that describe your UI.

When Jetpack Compose runs your composables for the first time, during initial composition, it will keep track of the composables that you call to describe your UI in a Composition. Then, when the state of your app changes, Jetpack Compose schedules a recomposition. Recomposition is when Jetpack Compose re-executes the composables that may have changed in response to state changes, and then updates the Composition to reflect any changes.

A Composition can only be produced by an initial composition and updated by recomposition. The only way to modify a Composition is through recomposition.

## Dynamic content

```kotlin
@Composable
fun MoviesScreen(movies: List<Movie>) {
    Column {
        for (movie in movies) {
            // MovieOverview composables are placed in Composition given its
            // index position in the for loop
            MovieOverview(movie)
        }
    }
}
```

In the example above, Compose uses the execution order in addition to the call site to keep the instance distinct in the Composition. If a new movie is added to the bottom of the list, Compose can reuse the instances already in the Composition since their location in the list haven't changed and therefore, the movie input is the same for those instances.

However, if the movies list changes by either adding to the top or the middle of the list, removing or reordering items, it'll cause a recomposition in all MovieOverview calls whose input parameter has changed position in the list. That's extremely important if, for example, MovieOverview fetches a movie image using a side effect. If recomposition happens while the effect is in progress, it will be cancelled and will start again.

Ideally, we want to think of the identity of the MovieOverview instance as linked to the identity of the movie that is passed to it. If we reorder the list of movies, ideally we would similarly reorder the instances in the Composition tree instead of recomposing each MovieOverview composable with a different movie instance. Compose provides a way for you to tell the runtime what values you want to use to identify a given part of the tree: the key composable.

By wrapping a block of code with a call to the key composable with one or more values passed in, those values will be combined to be used to identify that instance in the composition. The value for a key does not need to be globally unique, it needs to only be unique amongst the invocations of composables at the call site. So in this example, each movie needs to have a key that's unique among the movies; it's fine if it shares that key with some other composable elsewhere in the app.

```kotlin
@Composable
fun MoviesScreenWithKey(movies: List<Movie>) {
    Column {
        for (movie in movies) {
            key(movie.id) { // Unique ID for this movie
                MovieOverview(movie)
            }
        }
    }
}
```

Some composables have built-in support for the key composable. For example, LazyColumn accepts specifying a custom key in the items DSL.

```kotlin
@Composable
fun MoviesScreenLazy(movies: List<Movie>) {
    LazyColumn {
        items(movies, key = { movie -> movie.id }) { movie ->
            MovieOverview(movie)
        }
    }
}
```

## Skipping if the inputs haven't changed

During recomposition, some eligible composable functions can have their execution be skipped entirely if their inputs have not changed from the previous composition.

A composable function is eligible for skipping unless:
- The function has a non-Unit return type
- The function is annotated with `@NonRestartableComposable` or `@NonSkippableComposable`
- A required parameter is of a non-stable type

There is an experimental compiler mode, Strong Skipping, which relaxes the last requirement.

In order for a type to be considered stable it must comply with the following contract:
- The result of equals for two instances will forever be the same for the same two instances.
- If a public property of the type changes, Composition will be notified.
- All public property types are also stable.

There are some important common types that fall into this contract that the Compose compiler will treat as stable, even though they are not explicitly marked as stable by using the @Stable annotation:
- All primitive value types: Boolean, Int, Long, Float, Char, etc.
    Strings
- All function types (lambdas)

All of these types are able to follow the contract of stable because they are immutable. Since immutable types never change, they never have to notify Composition of the change, so it is much easier to follow this contract.

> 🌟 Note: All deeply immutable types can safely be considered stable types.

One notable type that is stable but is mutable is Compose’s MutableState type. If a value is held in a MutableState, the state object overall is considered to be stable as Compose will be notified of any changes to the .value property of State.

When all types passed as parameters to a composable are stable, the parameter values are compared for equality based on the composable position in the UI tree. Recomposition is skipped if all the values are unchanged since the previous call.
Key Point: Compose skips the recomposition of a composable if all the inputs are stable and haven't changed. The comparison uses the equals method.

Compose considers a type stable only if it can prove it. For example, an interface is generally treated as not stable, and types with mutable public properties whose implementation could be immutable are not stable either.

If Compose is not able to infer that a type is stable, but you want to force Compose to treat it as stable, mark it with the @Stable annotation.

```kotlin
// Marking the type as stable to favor skipping and smart recompositions.
@Stable
interface UiState<T : Result<T>> {
    val value: T?
    val exception: Throwable?

    val hasError: Boolean
        get() = exception != null
}
```

In the code snippet above, since UiState is an interface, Compose could ordinarily consider this type to be not stable. By adding the `@Stable` annotation, you tell Compose that this type is stable, allowing Compose to favor smart recompositions. This also means that Compose will treat all its implementations as stable if the interface is used as the parameter type.

> 💡 Key Point: If Compose is not able to infer the stability of a type, annotate the type with @Stable to allow Compose to favor smart recompositions.