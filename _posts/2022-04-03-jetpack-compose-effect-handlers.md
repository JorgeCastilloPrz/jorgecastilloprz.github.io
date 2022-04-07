---
layout: post
current: post
cover: assets/images/kyoto3.jpeg
navigation: True
title: Jetpack Compose Effect Handlers
date: 2022-04-03 10:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Learn how to run your side effects 🌀 bound to the `@Composable` lifecycle.

## What is a side effect? 🌀

Any Android application contains side effects, which are also very frequently called "effects", in case you were wondering. A side effect is essentially **anything that escapes the scope of the function** where it is called.

Here is an example of a side effect to keep an external state updated.

> ⚠️ Please **don't do this**, this is an anti-pattern in Compose that actually implies different issues that I'll explain below.

```kotlin
@Composable
fun MyScreen(drawerTouchHandler: TouchHandler) {
  val drawerState = rememberDrawerState(DrawerValue.Closed)

  drawerTouchHandler.enabled = drawerState.isOpen
  // ...
}
```

This composable describes a screen with a drawer with touch handling support. The drawer state is initialized as `Closed`, but might change to `Open` over time. In the initial composition and every later recomposition, the Composable function will execute itself and therefore notify the `TouchHandler` about the current drawer state.

Line `drawerTouchHandler.enabled = drawerState.isOpen` is an actual side effect. We're initializing a callback reference on an external object **as a side effect of the composition**.

## Why is this a red flag? 🚩

When doing it right in the `@Composable` function body, **we don't have any control over when the effect runs**. Composable functions must be restartable and idempotent (always yield the same state), since they might re-execute multiple times due to recomposition (among other reasons). For this reason we must never rely on the composition as a means to trigger a side effect. The effect from above will run on every composition / recomposition, and will **never get disposed**, opening the door to accumulative leaks.

> Composable functions are restartable and idempotent. That means they might **execute multiple times**.

A side effect of the composition could also be a **network or a database request**, for example. Imagine we need to load the data to display on screen from a network service. What would happen if the Composable recomposes multiple times? Should we run multiple network requests / queries? Or one across all recompositions? 🤔 Also, what if the composable leaves the composition before the request completes? We might prefer cancelling the job at that point, right?

We definitely need to keep the effect under control.

> In the future, compositions could be potentially **offloaded to different threads**, executed in parallel, in different order, or similar things. That's a door for diverse potential optimizations that the Jetpack Compose team wants to keep open, and that is also why Composable functions must be absolutely independent of each other.

## What we need 🤔

Overall, we need mechanisms for ensuring that:

* Effects run on the correct composable lifecycle step. Not too early, not too late. That must be after the Composable enters the Composition, and before it leaves.
* Suspend effects run on a conveniently configured runtime (Coroutine).
* Effects that capture references have their chance to dispose those when leaving composition.
* Ongoing suspend effects are cancelled when leaving composition.
* Effects that depend on an input that varies over time are automatically disposed / cancelled and relaunched every time it varies.

These mechanisms are provided by the Jetpack Compose **Effect handlers** 💫

## Effect Handlers 👀

Before describing them let me give you a sneak peek on the `@Composable` lifecycle, since that will be relevant from this point onwards.

Any Composable enters the composition when attached to the UI tree, and finally leaves the composition when detached from it. Between both events, effects might run. Some effects can outlive this cycle, so you can span an effect **across recompositions**.

This is all we need to know for now, let's keep moving 🏃‍♂️

I thought we could divide effects in two categories for clarity:

* **Non-suspend effects** 👉 E.g: Run a side effect to initialize a callback when the Composable enters the composition, dispose it when it leaves.
* **Suspend effects** 👉 E.g: Load data from network to feed some UI state.

## Non-suspend effects

### DisposableEffect

Side effect of the composition lifecycle that requires a cleanup before leaving.

* Used for non suspended effects that **require being disposed**.
* Fired the first time (when composable enters composition) and then every time its keys change.
* Requires `onDispose` callback at the end. It is disposed when the composable leaves the composition, and also on every recomposition when its keys have changed. In that case, the effect is disposed and relaunched.

```kotlin
@Composable
fun backPressHandler(onBackPressed: () -> Unit, enabled: Boolean = true) {
  val dispatcher = LocalOnBackPressedDispatcherOwner.current?.onBackPressedDispatcher

  val backCallback = remember {
    object : OnBackPressedCallback(enabled) {
      override fun handleOnBackPressed() {
        onBackPressed()
      }
    }
  }

  DisposableEffect(dispatcher) { // dispose/relaunch if dispatcher changes
    dispatcher?.addCallback(backCallback)
    onDispose {
      backCallback.remove() // avoid leaks!
    }
  }
}
```

Here we have a back press handler that attaches a callback to a dispatcher obtained from a `CompositonLocal`. We want to attach the callback when the composable enters the composition, but **also when the dispatcher varies**. To achieve that, we can **pass the dispatcher as the effect handler key**. That'll make sure the effect is disposed and relaunched in that case.

Callback is also disposed when the composable finally leaves the composition.

If you'd rather only running the effect once when entering the composition, span the same effect across recompositions, and dispose it when leaving, you could **pass a constant key**: `DisposableEffect(true)` or `DisposableEffect(Unit)`. Since the key never varies, it will not cancel and relaunch.

> `DisposableEffect` always requires at least one key.

### SideEffect

Another side effect of the composition. This one is a bit special since it's like a "fire on this composition or forget". If the composition fails for any reason, it is **discarded**. It is used for notifying about state updates to objects not managed by Compose.

If you are a bit familiar with the internals of the Compose runtime, note that it's an effect **not stored in the slot table**, meaning it does not outlive the composition, and it will not get retried in the future across recompositions or anything like that.

* Used for effects that **do not require disposing**.
* Runs after every **successful** composition / recomposition.
* Useful to **publishing updates to external states**.

```kotlin
@Composable
fun MyScreen(drawerTouchHandler: TouchHandler) {
  val drawerState = rememberDrawerState(DrawerValue.Closed)

  SideEffect {
    drawerTouchHandler.enabled = drawerState.isOpen
  }

  // ...
}
```

This fixes the anti-pattern we shared in the beginning. Here we also notify about the drawer state on every single composition or recomposition, but we do it **only for the successful ones**. Also, if the `TouchHandler` was a singleton living during the complete application execution because this was our main screen (always visible), we might not want to dispose the reference at all.

> Use `SideEffect` to **publish updates** to some external state not managed by the compose `State` system to keep it always on sync.

### currentRecomposeScope

This is more an effect itself than an effect handler, but it's interesting to cover.

As an Android dev you might be familiar with the `View` system `invalidate` counterpart, which essentially enforces a new measuring, layout and drawing passes on your view. It was heavily used to create frame based animations using the `Canvas`, for example. On every drawing tick you'd invalidate the view and therefore draw again based on some elapsed time.

The `currentRecomposeScope` is an interface with a single purpose:

```kotlin
interface RecomposeScope {
    /**
     * Invalidate the corresponding scope, requesting the composer to
     * recompose this scope.
     */
    fun invalidate()
}
```

The `currentRecomposeScope` can be used from any Composable function, and you can call `currentRecomposeScope.invalidate()` to invalidate the composition locally 👉  proactively **enforce recomposition**.

> ⚠️ This is an anti-pattern in Jetpack Compose, since it is always suggested to rely on Compose snapshot state to drive recomposition as possible. Use it very sparingly.

It can be useful when using a source of truth that is **not a compose State** snapshot.

```kotlin
interface Presenter {
  fun loadUser(after: @Composable () -> Unit): User
}

@Composable
fun MyComposable(presenter: Presenter) {
  val user = presenter.loadUser { currentRecomposeScope.invalidate() } // not a State!

  Text("The loaded user: ${user.name}")
}
```

Here we have a presenter and we manually invalidate to enforce recomposition when there's a result, since we're not relying on Compose `State` here. This is obviously a very edgy situation, so you'll likely prefer leveraging `State` and smart recomposition the big majority of the time.

> For frame based animations Compose provides APIs to suspend and await until the next rendering frame on the choreographer. Then execution resumes and you can update some state with the elapsed time or whatever leveraging smart recomposition one more time. I suggest reading [the official animation docs](https://developer.android.com/jetpack/compose/animation#targetbasedanimation) for a better understanding.

### derivedStateOf

There are times when we need to have some state derived from other state objects.

* Useful when we have some original state that might vary over time, and then other states that need to be recalculated when the original one changes.
* Imagine we have a computation heavy calculation that is remembered and it depends on some other snapshot state. We'd likely want it to be recalculated only when the original state changes.
* This avoids recalculating on every recomposition.

This example is extracted from the official docs:

```kotlin
@Composable
fun TodoList(highPriorityKeywords: List<String> = listOf("Review", "Unblock", "Compose")) {

  val todoTasks = remember { mutableStateListOf<String>() }

  // Calculate high priority tasks only when the todoTasks or highPriorityKeywords
  // change, not on every recomposition
  val highPriorityTasks by remember(highPriorityKeywords) {
      derivedStateOf { todoTasks.filter { it.containsWord(highPriorityKeywords) } }
  }

  Box(Modifier.fillMaxSize()) {
      LazyColumn {
          items(highPriorityTasks) { /* ... */ }
          items(todoTasks) { /* ... */ }
      }
      /* Rest of the UI where users can add elements to the list */
  }
}
```

This makes `highPriorityTasks` recalculate only when `todoTasks` changes. Note how a key is passed to `remember`. This will make it execute again when the key changes, and a new derived state will be created to replace the old one.

## Suspended effects

### rememberCoroutineScope

This call creates and remembers a `CoroutineScope` used to create jobs that can be thought as children of the composition.

* Used to run **suspended effects bound to the composition lifecycle**.
* Creates `CoroutineScope` bound to this composition lifecycle.
* The scope is **cancelled when leaving the composition**.
* Same scope is returned across compositions, so we can keep submitting more tasks to it and all ongoing ones will be cancelled when finally leaving.
* Useful to launch jobs **in response to user interactions**.
* Runs the effect on the applier dispatcher (Usually [`AndroidUiDispatcher.Main`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUiDispatcher.android.kt)) when entering.

```kotlin
@Composable
fun SearchScreen() {
  val scope = rememberCoroutineScope()
  var currentJob by remember { mutableStateOf<Job?>(null) }
  var items by remember { mutableStateOf<List<Item>>(emptyList()) }

  Column {
    Row {
      TextField("Start typing to search",
        onValueChange = { text ->
          currentJob?.cancel()
          currentJob = scope.async {
            delay(1000)
            items = viewModel.search(query = text)
          }
        }
      )
    }
    Row { ItemsVerticalList(items) }
  }
}
```

This is a throttling on the UI side. You might have done this in the past using `postDelayed` or a `Handler`. Every time a text input changes we want to cancel any previous ongoing jobs, and post a new one with a delay, so we always enforce a minimum delay between potential network requests, for example.

> The difference with `LaunchedEffect` is that `LaunchedEffect` is used for scoping jobs initiated by the composition, while `rememberCoroutineScope` is thought for scoping jobs **initiated by a user interaction**.
>
> In fact, `LaunchedEffect` is a Composable function, so you can only run it in a Composable's body, while rememberCoroutineScope.launch (or async) could be used outside of a Composable (e.g: a click listener).

### LaunchedEffect

This is the suspend variant for loading the initial state of a Composable, as soon as it enters the composition.

* Runs the effect when entering the composition.
* Cancels the effect when leaving the composition.
* Cancels and relaunches the effect when key/s change/s.
* Also useful to **span a job across recompositions**.
* Runs the effect on the applier dispatcher (Usually [`AndroidUiDispatcher.Main`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidUiDispatcher.android.kt)) when entering.

```kotlin
@Composable
fun SpeakerList(eventId: String) {
  var speakers by remember { mutableStateOf<List<Speaker>>(emptyList()) }
  LaunchedEffect(eventId) { // cancelled / relaunched when eventId varies
    speakers = viewModel.loadSpeakers(eventId) // suspended effect
  }

  ItemsVerticalList(speakers)
}
```

Not much to say. The effect runs once when entering then once again every time the key varies, since our effect depends on its value. `onDispose` is not available here, since it runs a `suspend` effect, and that is already disposable (cancellable) by design. In this example the effect will get cancelled when leaving the composition, if it didn't complete yet.

> Remember that it's also cancelled every time the key varies. `LaunchedEffect` **requires at least one key**.

### produceState

This is actually syntax sugar built on top of `LaunchedEffect`.

* Use it when your `LaunchedEffect` ends up feeding a Compose `State` (which is essentially the majority of the cases).
* Relies on `LaunchedEffect`.

```kotlin
@Composable
fun SpeakersScreen(eventId: String) {
  val uiState = produceState(initialValue = emptyList<Speaker>(), eventId) {
    value = someSuspendEffect()
  }

  ItemsVerticalList(uiState.value)
}
```

You can provide a default value for the state, and also **one or multiple keys**. As you probably guessed, it will cancel the suspend effect and relaunch it whenever any of the keys change.

The only gotcha is that `produceState` allows to not pass any key, and in that case it will call `LaunchedEffect` with `Unit` as the key, making the effect **span across compositions**. Keep that in mind since the API surface does not make it explicit.

### rememberUpdatedState

This is not an effect handler per se, but is used in combination with those. Sometimes we want to capture a value in our effect and be able to update it later, but we **don't want to trigger a restart when it changes**.

* Useful for effects for long lived operations that would be costly to recreate and restart.
* Useful when the effect depends on a value that might change over time but we still want the effect to span across recompositions.
* Frequently used in combination with `LaunchedEffect`.

This example is extracted from the [official docs](https://developer.android.com/jetpack/compose/side-effects) 👇

```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {

  // This will always refer to the latest onTimeout function that
  // LandingScreen was recomposed with
  val currentOnTimeout by rememberUpdatedState(onTimeout)

  // Create an effect that matches the lifecycle of LandingScreen.
  // If LandingScreen recomposes, the delay shouldn't start again.
  LaunchedEffect(true) {
    delay(SplashWaitTimeMillis)
    currentOnTimeout()
  }

  /* Landing screen content */
}
```

What `rememberUpdatedState` does is wrap the value into `mutableStateOf` and `remember` it, so it becomes Compose observable snapshot state, but on top of that, every time it is called it will also update its value. So, on every recomposition it will read the new value (that might have changed) and update the captured state. This will not retrigger the effect, though.

## Third party library adapters

We frequently need to consume other data types from third party libraries like `Observable`, `Flowable`, `Single`, `Maybe` and `Completable` from RxJava (2 and 3), `Flow` from KotlinX Coroutines, or `LiveData`. Jetpack Compose provides adapters for the most frequent third party types, so depending on the library you'll need to fetch a different dependency:

```groovy
implementation "androidx.compose.runtime:runtime-livedata:$compose_version" // LiveData
implementation "androidx.compose.runtime:runtime-rxjava2:$compose_version" // RxJava 2
implementation "androidx.compose.runtime:runtime-rxjava3:$compose_version" // RxJava 3
implementation "androidx.compose.runtime:runtime:$compose_version" // includes Flow adapters
```

**All those adapters end up delegating to the effect handlers**. All of them attach an observer using the third party library apis, and end up mapping every emitted element to an ad hoc `MutableState` that is exposed by the adapter function as an immutable `State`.

Some examples for the different libraries below 👇

### LiveData

```kotlin
class MyComposableVM : ViewModel() {
  private val _user = MutableLiveData(User("John"))
  val user: LiveData<User> = _user
  //...
}

@Composable
fun MyComposable(viewModel: MyComposableVM = viewModel()) {

  val user by viewModel.user.observeAsState()

  Text("Username: ${user?.name}")
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime-livedata/src/main/java/androidx/compose/runtime/livedata/LiveDataAdapter.kt;l=54?q=observeAsState) is the actual implementation of `observeAsState` which relies on `DisposableEffect` to hook an observer to the `LiveData` that will update the resulting State value every time a new value is emitted. It will also dispose this observer when the effect leaves the composition.

### RxJava2 and RxJava3

```kotlin
class MyComposableVM : ViewModel() {
  val user: Observable<ViewState> = Observable.just(ViewState.Loading)
  //...
}

@Composable
fun MyComposable(viewModel: MyComposableVM = viewModel()) {
  val uiState by viewModel.user.subscribeAsState(ViewState.Loading)

  when (uiState) {
    ViewState.Loading -> TODO("Show loading")
    ViewState.Error -> TODO("Show Snackbar")
    is ViewState.Content -> TODO("Show content")
  }
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime-rxjava2/src/main/java/androidx/compose/runtime/rxjava2/RxJava2Adapter.kt;l=49?q=subscribeAsState&sq=) is the implementation for `susbcribeAsState()` for RxJava 2, and [here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime-rxjava3/src/main/java/androidx/compose/runtime/rxjava3/RxJava3Adapter.kt;l=50?q=subscribeAsState&sq=) is the one for RxJava 3. Same story 🙂 both of them rely on `DisposableEffect` to subscribe to the observable type and map every new item emitted to the resulting Compose State. It will dispose the subscription when the effect leaves the composition.

All the observable types from both libraries are supported.

### KotlinX Coroutines Flow

```kotlin
class MyComposableVM : ViewModel() {
  val user: Flow<ViewState> = flowOf(ViewState.Loading)
  //...
}

@Composable
fun MyComposable(viewModel: MyComposableVM = viewModel()) {
  val uiState by viewModel.user.collectAsState(ViewState.Loading)

  when (uiState) {
    ViewState.Loading -> TODO("Show loading")
    ViewState.Error -> TODO("Show Snackbar")
    is ViewState.Content -> TODO("Show content")
  }
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotFlow.kt;l=60?q=collectAsState) is the implementation for `collectAsState`, part of the Compose runtime library. This one relies on `produceState` instead which delegates on `LaunchedEffect`. It collects the flow in the current `CoroutineContext` if no explicit context is provided, otherwise it will collect in the context we pass to it.

For converting Compose State into a cold Flow (the opposite integration), there is also `snapshotFlow` (example extracted from the official docs):

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
  // ...
}

LaunchedEffect(listState) {
  snapshotFlow { listState.firstVisibleItemIndex }
    .map { index -> index > 0 }
    .distinctUntilChanged()
    .filter { it == true }
    .collect {
      MyAnalyticsService.sendScrolledPastFirstItemEvent()
    }
}
```

This will create a `Flow` from the observable snapshot State. [Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotFlow.kt;l=110?q=snapshotFlow&sq=) is the implementation. It will read any snapshot state changes in the block passed to `snapshotFlow {}`, and emit them in the resulting `Flow`.

So, as you can see most of these adapters rely on the effect handlers explained in this post, and you could easily write your own adapters for other libraries or data types following the same pattern.

## Final thoughts

Any apps have side effects that we want to keep under control and make Composable lifecycle aware. By wrapping them with a convenient effect handler we make sure it runs in the proper lifecycle step, has its chance to get disposed or cancelled when required to avoid leaks, re-executes itself when its inputs vary, and runs in a convenient context (CoroutineContext) when required.

### Learn more about this

This is one of the topics covered in detail in a chapter about effect handlers of the [Jetpack Compose Internals](https:leanpub.com/composeinternals) book. Give it a read if you want to know more about effect handlers, how they work internally, and how they are represented / handled by the Compose runtime.

<img width="300" src="../assets/images/title_page.png"/>

### Where you can find me

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
