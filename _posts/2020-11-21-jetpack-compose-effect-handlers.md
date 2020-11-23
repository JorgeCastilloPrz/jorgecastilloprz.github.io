---
layout: post
current: post
cover: assets/images/kyoto3.jpeg
navigation: True
title: Jetpack Compose Effect Handlers
date: 2020-11-21 10:00:00
tags: [kotlin, android]
class: post-template
subclass: 'post'
author: jorge
---

Learn how to run your side effects 🌀 bound to the `@Composable` lifecycle.

## What is a side effect? 🌀

Any Android applications contain side effects. They are also called "effects" quite often, in case you've been wondering. A side effect is essentially **anything that escapes the scope of the function**.

Here is an example of what could be a side effect to keep an external state updated. Please **don't do this**, this is an anti-pattern in Compose that actually implies different issues I'll explain below.

```kotlin
@Composable
fun MyScreen(drawerTouchHandler: TouchHandler) {
  val drawerState = rememberDrawerState(DrawerValue.Closed)

  drawerTouchHandler.enabled = drawerState.isOpen

  // ...
}
```

This composable describes a screen with a drawer with touch handling support. The drawer state is initialized as `Closed`, but might change to `Open` over time. For every composition and recomposition, the composable notifies the `TouchHandler` about the current drawer state to enable touch handling support only when it's `Open`.

Line `drawerTouchHandler.enabled = drawerState.isOpen` is an actual side effect. We're initializing a callback reference on an external object as a **side effect of the composition**.

The problem on doing it right in the `@Composable` function body is that we don't have any control on when this runs, so it'll run on every composition / recomposition, and **never disposed**, opening the door to potential leaks.

Remember `@Composable` functions are prepared by the Compose compiler to be restartable and idempotent. That means they might **run multiple times**.

A side effect of the composition could also be a **network or a database request**, for example. Imagine we need to load the data to display on screen from a network service. What would happen if the composable leaves the composition before it completes?. We might prefer cancelling the job at that point, right?

## What we need 🤔

In the future, compositions could be potentially **offloaded to different threads**, executed in parallel, in different order, or similar things. That's a door for diverse potential optimizations the Compose team wants to keep open, and that is also why you'd never want to run your side effects right away during the composition without any sort of control.

Overall, we need mechanisms for making sure that:

* Effects run on the correct composable lifecycle step. Not too early, not too late. Just when the composable is ready for it.
* Suspended effects run on a conveniently configured runtime (Coroutine and convenient `CoroutineContext`).
* Effects that capture references have their chance to dispose those when leaving composition.
* Ongoing suspended effects are cancelled when leaving composition.
* Effects that depend on an input that varies over time are automatically disposed / cancelled and relaunched every time it varies.

These mechanisms are provided by Jetpack Compose and called **Effect handlers**.


## Disclaimer ⚠️

Effect handlers in Compose are under heavy development iterations. They have changed names many times in the latest alphas. Keep in mind they might still vary a bit more in the near future. All the effect handlers shared on this post are the ones available as of today for latest `1.0.0-SNAPSHOT`.

## Effect Handlers 👀

Before describing them let me give you a sneak peek on the `@Composable` lifecycle, since that'll be relevant from this point onwards.

Any composable enters the composition (**onEnter**) when materialized on screen, and finally leaves the composition (**onLeave**) when removed from the UI tree. Between both events, effects might run. Some effects can outlive the composable lifecycle, so you can span an effect across compositions.

This is all we need to know for now, let's keep moving 🏃‍♂️

We could divide effect handlers in two categories:

* **Non suspended effects** 👉 E.g: Run a side effect to initialize a callback when the Composable enters the composition, dispose it when it leaves.
* **Suspended effects** 👉 E.g: Load data from network to feed some UI state.

## Non suspended effects

### DisposableEffect

This is the old old onCommit + onDispose combo. A side effect of the composition lifecycle.

* Used for effects that **require being disposed**.
* Fired the first time (when composable enters composition) and then every time its subjects change.
* Requires a dispose callback at the end. It is disposed when the composable leaves the composition, and also on every recomposition when its subjects have changed. In that case, the effect is disposed and relaunched.

```kotlin
@Composable
fun backPressHandler(onBackPressed: () -> Unit, enabled: Boolean = true) {
  val dispatcher = BackPressedDispatcherAmbient.current.onBackPressedDispatcher

  val backCallback = remember {
      object : OnBackPressedCallback(enabled) {
          override fun handleOnBackPressed() {
              onBackPressed()
          }
      }
  }

  DisposableEffect(dispatcher) { // dispose/relaunch if dispatcher changes
    dispatcher.addCallback(backCallback)
    onDispose {
          backCallback.remove() // avoid leaks!
      }
  }
}
```

Here we have a back press handler that attaches a callback to a dispatcher obtained from an `Ambient`. We want to attach the callback when the composable enters the composition, and also when the dispatcher varies. To achieve that, we can **pass the dispatcher as the effect handler subject**. That'll make sure the effect is disposed and relaunched in that case.

Callback is also disposed when the composable finally leaves the composition.

If you only want to run the effect once **onEnter** and dispose it **onLeave**, you can pass a constant subject to it: `DisposableEffect(true)` or `DisposableEffect(Unit)`. That will mimic the old `onActive` and `onCommit(Unit)` behavior.

### SideEffect

Another side effect of the composition. This one is a bit special since it's like a "fire on this composition or forget". If the composition fails for any reason, it is **discarded**.

If you are a bit familiar with the internals of the Compose runtime, note that it's an effect **not stored in the slot table**, meaning it does not outlive the composition, and it will not get retried in future across compositions or anything like that.

* Used for effects that **do not require disposing**.
* Runs after every single composition / recomposition.
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

This is the same snippet we used in the beginning. Here we care about the current state of the drawer, which might vary at any point in time. In that sense, we need to notify it for every single composition or recomposition.  Also, if the `TouchHandler` was a singleton living during the complete application execution because this was our main screen (always visible), we might not want to dispose the reference at all.

### invalidate

This is more an effect itself than an effect handler, but it's interesting to cover.

As an Android dev you might be familiar with the `View` system `invalidate` counterpart, which essentially enforces a new measuring, layout and drawing passes on your view. It was heavily used to create frame based animations using the `Canvas`, for example. So on every drawing tick you'd invalidate the view and therefore draw again based on some elapsed time.

This variant is quite similar.

* Invalidates composition locally 👉 enforces recomposition.
* Useful when using a source of truth that is **not a compose State** snapshot.

```kotlin
interface Presenter {
    fun loadUser(after: @Composable () -> Unit): User
}

@Composable
fun MyComposable(presenter: Presenter) {
    val user = presenter.loadUser { invalidate() } // not a State!

    Text(text = "The loaded user: ${user.name}")
}
```

Here we have an old school presenter and we manually invalidate to enforce recomposition when there's a result, since we're not using `State` in any way. This is obviously a very edgy situation, so you'll likely prefer levearing `State` and smart recomposition the big majority of the time. Just know this is a thing in Compose.

So overall, ⚠️ Use sparingly! ⚠️. Use `State` for smart recomposition when it varies as possible, since that'll make sure to get the most out of the Compose runtime.

For frame based animations as the ones described above, Compose provides APIs to suspend and await until the next rendering frame on the choreographer, so at that point the execution resumes and you can update some state according to the elapsed time or whatever, one more time leveraging smart recomposition (making the Composable read from that state). So you'll not need `invalidate` for that either.

### rememberCoroutineScope

We start with the **suspending effect handlers** here. This call creates a `CoroutineScope` used to create jobs that can be thought as children of the composition.

* Used to run **suspended effects bound to the composition lifecycle**.
* Creates `CoroutineScope` bound to this composition lifecycle.
* The scope is **cancelled when leaving the composition**.
* Same scope is returned across compositions, so you can keep submitting more tasks to it and all ongoing ones can be cancelled when finally leaving.
* Useful to launch jobs **in response to user interactions**.
* Runs the effect on the applier dispatcher (Usually [`AndroidUiDispatcher.Main`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:compose/runtime/runtime-dispatch/src/androidMain/kotlin/androidx/compose/runtime/dispatch/AndroidUiDispatcher.kt;l=29;drc=773cdb49ea3e3fc440967a278973e3bd211beb21)) when entering.

```kotlin
@Composable
fun SearchScreen() {
  val scope = rememberCoroutineScope()
  var currentJob by remember { mutableStateOf<Job?>(null) }
  var items by remember { mutableStateOf<List<Item>>(emptyList()) }

  Column {
    Row {
      TextInput(
        afterTextChange = { text ->
          currentJob?.cancel()
          currentJob = scope.async {
            delay(threshold)
            items = viewModel.search(query = text)
          }
        }
      )
    }
    Row { ItemsVerticalList(items) }
  }
}
```

This is a throttling on the UI side. You might have done this in the past using `postDelayed` or a `Handler` with the `View` system. Every time a text input changes we want to cancel any previous ongoing jobs, and post a new one with a delay, so we always enforce a minimum delay between potential network requests, for example.

Note how we remember the `CoroutineScope`, and use it to submit every new job to it. That'll make sure those jobs are cancelled if required when the composable finally leaves the composition.

Note how we also retain a reference to the jobs so we can manually cancel them before launching a new one.

### LaunchedEffect

This is the suspending variant for loading the initial state of a Composable, as soon as it enters the composition.

* Runs the effect when entering the composition.
* Cancels the effect when leaving the composition.
* Cancels and relaunches the effect when subject/s change/s.
* Useful to **span a job across recompositions**.
* Runs the effect on the applier dispatcher (Usually [`AndroidUiDispatcher.Main`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:compose/runtime/runtime-dispatch/src/androidMain/kotlin/androidx/compose/runtime/dispatch/AndroidUiDispatcher.kt;l=29;drc=773cdb49ea3e3fc440967a278973e3bd211beb21)) when entering.

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

Not much to say. The effect runs once **onEnter** then once again every time `eventId` varies, since our effect depends on its value. It'll get cancelled when leaving the composition.

Remember that it's also cancelled every time it needs to be relaunched.

### produceState

This is actually syntax sugar built on top of `LaunchedEffect`.

* Used when your `LaunchedEffect` ends up feeding a `State` (which is most of the time).
* Relies on `LaunchedEffect`.

```kotlin
@Composable
fun SearchScreen(eventId: String) {
  val uiState = produceState(initialValue = emptyList<Speaker>(), eventId) {
    viewModel.loadSpeakers(eventId) // suspended effect
  }

  ItemsVerticalList(uiState.value)
}
```

You can provide a default value for the state, and also **one or multiple subjects**.

## Third party library adapters

We frequently need to consume other data types from third party libraries like `Observable`, `Flow`, or `LiveData`. Jetpack Compose provides adapters for the most frequent third party types, so depending on the library you'll need to fetch a different dependency:

```groovy
implementation "androidx.compose.runtime:runtime-livedata:$compose_version"
implementation "androidx.compose.runtime:runtime-rxjava2:$compose_version"
implementation "androidx.compose.runtime:runtime-flow:$compose_version"
```

**All those adapters end up delegating on the effect handlers**. All of them attach an observer using the third party library apis, and end up mapping every emitted element to an ad hoc `MutableState` that is exposed by the adapter function as an immutable `State`.

Some examples for the different libraries below 👇

### LiveData

```kotlin
class MyComposableVM : ViewModel() {
    private val _user = MutableLiveData(User("John"))
    val user: LiveData<User> = _user
    //...
}

@Composable
fun MyComposable() {
    val viewModel = viewModel<MyComposableVM>()

    val user by viewModel.user.observeAsState()

    Text("Username: ${user?.name}")
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:compose/runtime/runtime-livedata/src/main/java/androidx/compose/runtime/livedata/LiveDataAdapter.kt;l=55?q=LiveDataAdapter) is the actual implementation of `observeAsState` which relies on `onCommit / onDispose` handlers. I bet that will likely be replaced by `DisposableEffect` at some point.

### RxJava2

```kotlin
class MyComposableVM : ViewModel() {
    val user: Observable<ViewState> = Observable.just(ViewState.Loading)
    //...
}

@Composable
fun MyComposable() {
    val viewModel = viewModel<MyComposableVM>()

    val uiState by viewModel.user.subscribeAsState(ViewState.Loading)

    when (uiState) {
        ViewState.Loading -> TODO("Show loading")
        ViewState.Error -> TODO("Show Snackbar")
        is ViewState.Content -> TODO("Show content")
    }
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:compose/runtime/runtime-rxjava2/src/main/java/androidx/compose/runtime/rxjava2/RxJava2Adapter.kt;l=130) is the implementation for `susbcribeAsState()`. Same story 🙂The same extension is also available for `Flowable`.

### KotlinX Coroutines Flow

```kotlin
class MyComposableVM : ViewModel() {
    val user: Flow<ViewState> = flowOf(ViewState.Loading)
    //...
}

@Composable
fun MyComposable() {
    val viewModel = viewModel<MyComposableVM>()

    val uiState by viewModel.user.collectAsState(ViewState.Loading)

    when (uiState) {
        ViewState.Loading -> TODO("Show loading")
        ViewState.Error -> TODO("Show Snackbar")
        is ViewState.Content -> TODO("Show content")
    }
}
```

[Here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/FlowAdapter.kt;l=55?q=FlowAdapter&ss=androidx%2Fplatform%2Fframeworks%2Fsupport) is the implementation for `collectAsState`. This one is a bit different since `Flow` needs to be consumed from a suspended context. That is why it relies on `produceState` instead which delegates on `LaunchedEffect`.

So, as you can see all these adapters rely on the effect handlers explained in this post, and you could easily write your own. Like for example this one for Arrow Streams (drain being the suspend operation to consume Arrow Streams):

```kotlin
@Composable
fun <T> Stream<T>.drainAsState(
  initial: T
): State<T> {
  val state = remember { mutableStateOf(initial) }

  LaunchedArrowEffect {
    evalOn(ComputationPool) {
      this@drainAsState.effectTap { state.value = it }.drain()
    }
  }
  return state
}
```


## Final thoughts and talk slides!

Those are probably the most relevant effect handlers I've found in the sources. Feel free to ask about / suggest different ones 🙏

All this exploration was done via [cs.android.com](https://cs.android.com/), which is a really convenient tool when digging into sources 👍

Any apps have side effects which we don't want to run right away in the plain `@Composable` body, but keep bounded to the composable lifecycle. By wrapping them with the convenient effect handler we make sure it runs in the proper lifecycle step, has its chance to get released or cancelled to avoid leaks, and runs in a convenient context (CoroutineContext) provided by the effect handler when required.

This was part of a talk I gave in [Droidcon Americas](https://www.online.droidcon.com/americas2020) about Jetpack Compose. Here you have the slides! 🙌

<script async class="speakerdeck-embed" data-id="85c6d1852e494c0ebb57ac45a5406e84" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

---

You might be interested in other posts I wrote about Jetpack Compose:

* [Awaiting next frame](https://jorgecastillo.dev/jetpack-compose-await-next-frame)
* [Compose ViewPager](https://jorgecastillo.dev/compose-viewpager)
* [Compose ConstraintLayout](https://jorgecastillo.dev/jetpack-compose-constraintlayout)
* [Compose measuring and WithConstraints](https://jorgecastillo.dev/jetpack-compose-withconstraints)

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

More interesting stuff to come 🙌
