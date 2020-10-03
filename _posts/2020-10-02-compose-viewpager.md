---
layout: post
current: post
cover: assets/images/painting7.png
navigation: True
title: Jetpack Compose ViewPager
date: 2020-10-02 11:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Let's use a few minutes to learn how to write a swipeable pager composable.

## 🕵️‍♀️ The case

We are covering this [custom Pager implementation](https://github.com/android/compose-samples/blob/1630f6b35ac9e25fb3cd3a64208d7c9afaaaedc5/Jetcaster/app/src/main/java/com/example/jetcaster/util/Pager.kt#L130) from the Compose JetCaster official sample, so all the credit goes to the Googlers who created it. That said, we can use the chance to peek into it and learn how to write our own swipeable components.

Its behavior is not perfect if you compare it to its View based counterpart, but it's more than enough as a base for our exploration today. And I'm sure it'll keep improving 🙌

Also note that for the time being **there is not an official Pager composable** provided by Compose, even though there will probably be one in the future. This is just a guess, but these official samples also work great as a playground for the team to explore and test potential implementations in the open.

The Compose version used in this post is `1.0.0-alpha04`, as the one used in the official samples.

## 🤿 The immersion

Let's start by the composable declaration.

```kotlin
@Composable
fun Pager(
  state: PagerState,
  offscreenLimit: Int = 2,
  modifier: Modifier = Modifier,
  pageContent: @Composable PagerScope.() -> Unit
) {
  // ...
}
```

The composable accepts a few parameters (I'll leave the `PagerState` to the end since it's a bit more complex):

### offscreenLimit: Int

Like its counterpart for the Android `ViewPager`, it indicates the amount of non visible screens to be precomputed to either side of the current page. 

In this implementation, it will be used to know the number of pages to add to the composable **on every composition**. The secret is that the Pager will recompose every time the current page changes, so children are replaced by a new batch of pages every time.

### modifier: Modifier

The standard default `Modifier`, so the user can pass any desired modifiers, and then the internal implementation of this composable can also append it's own ones for achieving the required paging behavior.

### pageContent: @Composable PagerScope.() -> Unit

This is the composable lambda used for each child, or in other words, each screen in the Pager. The receiver is a `PagerScope`, which is just a class this composable will fill with relevant info for the user in real time, so it's accessible from the outside. Like current page, current offset or the current selection state. This way the user will be able to listen to those.

```kotlin
Pager(state = pagerState) {
  // currentPage, currentPageOffset, selectionState available in here!
  Image(
    ...
    modifier = Modifier.fillMaxSize()
  )
}
```

This "scope" pattern is very common in the compose world, when composables want to provide some contextual values to the outside when adding children.

### state: PagerState

This is the state our composable will observe and react to. To be more accurante, it's a composition of smaller `State`/s that will keep information about the current offset, maximum and minimum pages, and the current page selected. The `Pager` will recompose whenever any of those change.

Here is [the complete state implementation](https://github.com/android/compose-samples/blob/1630f6b35ac9e25fb3cd3a64208d7c9afaaaedc5/Jetcaster/app/src/main/java/com/example/jetcaster/util/Pager.kt#L47). It contains quite a bunch of code that I'm intentionally avoiding to paste here to not make it too overwhelming. Let's try to focus on the most relevant stuff and go step by step.

You'll see that it gets a `clock: AnimationClockObservable` parameter, that will be used for the animation. In Compose, animations work with a clock (for ticking), and some apis require to pass one explicitly. Here, we will use it to animate the `currentPageOffset`, which will be observed by the Pager composable to recompose accordingly. You can pass the `AnimationClockAmbient.current` from your composable lambda, if you want to. That one is the default clock used by the choreographer [for each rendering tick](https://developer.android.com/reference/kotlin/androidx/compose/animation/core/DefaultAnimationClock).

If you want to go further into each observable state definition here, you'll see things like:

```kotlin
private var _minPage by mutableStateOf(minPage)
  var minPage: Int
    get() = _minPage
    set(value) {
      _minPage = value.coerceAtMost(_maxPage)
      _currentPage = _currentPage.coerceIn(_minPage, _maxPage)
    }

private var _maxPage by mutableStateOf(maxPage)
  var maxPage: Int
    get() = _maxPage
    set(value) {
      _maxPage = value.coerceAtLeast(_minPage)
      _currentPage = _currentPage.coerceIn(_minPage, maxPage)
    }

private var _currentPage by mutableStateOf(currentPage.coerceIn(minPage, maxPage))
  var currentPage: Int
    get() = _currentPage
    set(value) {
      _currentPage = value.coerceIn(minPage, maxPage)
    }
```

`minPage`, `maxPage` and `currentPage` observable state declarations. You see how they're defined as private mutable states to be mutated from the inside, but exposed as public `Int` values to the outside, so out of this file they can only be consumed as standard properties. Real time observing is only needed for the Pager internals, i.e: this file.

Also note how they're linked, so every time `_minPage` or `_maxPage` states are updated, the `_currentPage` is also recalculated to be within the new limits.

Another state provided by this class is the current `selectionState`. There are a couple of potential selection states possible at any point in time for our composable:

```kotlin
enum class SelectionState { Selected, Undecided }

var selectionState by mutableStateOf(SelectionState.Selected)
```

The composable will move to `Undecided` meanwhile we are dragging or fling, and to `Selected` again when it gets idle (after completing a drag + fling animation, or if we select a page instantly with the function below 👇).

It also provides means to select a page proactively from user code. This function gives the chance to pass a `PagerState` scoped lambda to run during selection, in case there is any side effect you want to perform.

```kotlin
inline fun <R> selectPage(block: PagerState.() -> R): R = try {
  selectionState = SelectionState.Undecided
  block()
} finally {
  selectPage()
}

fun selectPage() {
  currentPage -= currentPageOffset.roundToInt()
  currentPageOffset = 0f
  selectionState = SelectionState.Selected
}
```

Note how the `currentPage` state is updated according to the `currentPageOffset`. This is because this `selectPage()` no args function is also called after manual interaction (when the fling animation after dragging ends), and depending on how much offset we currently have we might need to change the current page, or stay on the current one.

We'll learn a bit more on this when we peek into the composable code.

Here's the definition of the current offset state:

```kotlin
private var _currentPageOffset = AnimatedFloatModel(0f, clock = clock).apply {
    setBounds(-1f, 1f)
  }
  var currentPageOffset: Float
    get() = _currentPageOffset.value
    set(value) {
      val max = if (currentPage == minPage) 0f else 1f
      val min = if (currentPage == maxPage) 0f else -1f
      _currentPageOffset.snapTo(value.coerceIn(min, max))
    }
```

You will probably notice this one is a bit different. We don't see the `mutableStateOf()` call anywhere, but we do see an `AnimatedFloatModel`. In this case, this is also effectively an observable state, since [this class is a wrapper over a float value that is expected to change over time](https://developer.android.com/reference/kotlin/androidx/compose/animation/AnimatedFloatModel), and it exposes it as an **observable model** (following the old `@Model` naming convention by compose) so observers can recompose when it changes.

Note how it's constrained between -1 and 1, except for first and last pages. Here's a summary:

* **First page** 👉 allowed offset range: [-1, 0].
* **Last page** 👉 allowed offset range: [0, 1].
* **Any other page** 👉 allowed offset range: [-1, 1].

If we think about what that means visually (to grow a mental mapping), just imagine we can drag a pager in both directions. Here's how the offset updates accordingly:

| **Drag to left** | **Drag to right** |
|:----------:|:-------:|
| Goes negative (from 0 to -1) | Goes positive (from 0 to 1) |
|:----------:|:-------:|
| <img width="250" src="assets/images/pager_offset_drag_to_left.gif"/> | <img width="250" src="assets/images/pager_offset_drag_to_right.gif"/> |

Now you can see why first page only goes from 0 to -1, it can only be dragged in one direction since we're at the edge. Same thing happens for last page, which only goes from 0 to 1. For any other page you can drag in both directions, so offset ranges from -1 to 1.

Also note how the offset is always 0 when a page is selected (front and center, no animation ongoing, no offset).

The final utility this state class provides is the ability to fling and animate to center the current page once fling finishes. Fling animation is the one that happens by the inertia created by our manual drag when we release the finger. It depends on the current velocity:

```kotlin
fun fling(velocity: Float) {
    if (velocity < 0 && currentPage == maxPage) return
    if (velocity > 0 && currentPage == minPage) return

    _currentPageOffset.fling(velocity) { reason, _, _ ->
      if (reason != AnimationEndReason.Interrupted) {
        _currentPageOffset.animateTo(currentPageOffset.roundToInt().toFloat()) { _, _ ->
          selectPage()
        }
      }
    }
  }
```

We'll describe how to calculate velocity later, but see how it's constrained also, in case we're on the first or last page. That is because we don't want to apply fling animation if we already reached any of the edges. Velocity values will be signed integers, to indicate the acceleration direction.

By calling `currentPageOffset.fling(velocity)`, and given our `currentPageOffset` is an `AnimatedFloatModel` as described above, we can animate it using the required `velocity`, and it'll apply the required fling animation with its internally calculated deceleration and all that. 

The final lambda is a callback we pass to react when the fling animation ends. At that moment, this logic animates the current offset to it's closest int value, meaning it'll become a whole `Int`, reflecting the page number we want to snap to. 

Note the inertia after releasing the finger, that is the fling effect:

<img width="250" src="assets/images/pager_drag.gif"/> 

And this is all we wanted to know about the `PagerState` 🥳 This is great because we're now quite ready to understand how this composable works. It's finally the time to dive into it.

### The Pager ™️

[Here's the full code](https://github.com/android/compose-samples/blob/1630f6b35ac9e25fb3cd3a64208d7c9afaaaedc5/Jetcaster/app/src/main/java/com/example/jetcaster/util/Pager.kt#L130), but let me describe it step by step. You'll see that the composable structure looks something like this:

```kotlin
@Composable
fun Pager(
  state: PagerState,
  offscreenLimit: Int = 2,
  modifier: Modifier = Modifier,
  pageContent: @Composable PagerScope.() -> Unit
) {
  var pageSize by remember { mutableStateOf(0) }
  Layout(
    children = {
      // We'll create and add the children here (pages)
    },
    modifier = modifier.draggable(
      orientation = /* ... */,
      onDragStarted = { state.selectionState = PagerState.SelectionState.Undecided },
      onDragStopped = { velocity -> /* We'll start the fling here */ }
    ) { dx ->
      // We'll update the currentPageOffset based on the drag amount.
      }
    }
  ) { measurables, constraints ->
     // layouting (includes measuring and placing the elements)
    }
  }
}
```

So, a few things going on here. The first thing we need to realize is that this is written as a custom `Layout`, which essentially maps to how we created custom layouts in Android (remember the custom `ViewGroup`s). There are three steps required to create one:

* Provide the children.
* Measure the children.
* Place measured children according to the available space and constraints.

Let's keep that 3 steps process in mind.

The `pageSize` starts as `0` basically because it'll be computed within the measure block, which is the final lambda where we measure and place everything. This is like for Android Views, we cannot know the size of the composable or its pages too early (before the required measuring step).

Another relevant thing on the snippet above is the `draggable` modifier. We'll use that one to enable the dragging gesture, and to update the `currentPageOffset` state accordingly on drag.

Let's get in detail for each step.

### Adding the children

```kotlin
Layout(
    children = {
      val minPage = (state.currentPage - offscreenLimit).coerceAtLeast(state.minPage)
      val maxPage = (state.currentPage + offscreenLimit).coerceAtMost(state.maxPage)

      for (page in minPage..maxPage) {
        val pageData = PageData(page)
        val scope = PagerScope(state, page)
        key(pageData) {
          Box(alignment = Alignment.Center, modifier = pageData) {
            scope.pageContent()
          }
        }
      }
    },
    modifier = modifier.draggable(/* ... */)
  ) { measurables, constraints ->
    layout(constraints.maxWidth, constraints.maxHeight) {
      // layouting (includes measuring and placing the elements)
    }
  }
```

If we remember what we said on the previous section, the `offscreenLimit` passed will determine how many pages we can add, or in other words, precompute. So first two lines calculate the minimum and maximum pages based on that.

Then we loop from `minPage` to `maxPage` to add all the pages. Each page is wrapped and centered into a `Box`. 

For each page, some parent data is created for passing some information to the parent. This is just a requirement of the `key` wrapper used here to help the runtime differentiate all the pages in the loop for further smart recomposition. More details [in the official docs](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#key).

There is also the `PagerScope` we mentioned above to make some of the relevant information visible to the user on the call site. That will make the user able to access things like the current page, offset, or selection state.

### The draggable modifier

```kotlin
Layout(
    children = { /* ... */ },
    modifier = modifier = modifier.draggable(
      orientation = Orientation.Horizontal,
      onDragStarted = { state.selectionState = PagerState.SelectionState.Undecided },
      onDragStopped = { velocity ->
        // Velocity is in pixels per second, but we deal in percentage offsets, so we
        // need to scale the velocity to match
        state.fling(velocity / pageSize)
      }
    ) { dy ->
      with(state) {
        val pos = pageSize * currentPageOffset
        val max = if (currentPage == minPage) 0 else pageSize * offscreenLimit
        val min = if (currentPage == maxPage) 0 else -pageSize * offscreenLimit
        val newPos = (pos + dy).coerceIn(min.toFloat(), max.toFloat())
        currentPageOffset = newPos / pageSize
      }
    }
  ) { measurables, constraints ->
    layout(constraints.maxWidth, constraints.maxHeight) {
      // layouting (includes measuring and placing the elements)
    }
  }
```

It is configured to be `Horizontal` since this pager implementation doesn't support vertical dragging.

Whenever we start dragging, the `selectionState` is moved to `Undecided`.

When dragging stops we just need to call the `fling` function from the state we showcased on previous section. That one takes care of the fling animation, the snap animation after that, and finally updates the `currentPage`, resets the offset, and sets `selectionState` to `Selected`. 

Here, we only need to pass the current dragging velocity, which is provided by the modifier. We just need to make sure we scale it for a single page, since velocity by the modifier is calculated for the total composable measure.

The lambda block in the end is the side effect we do on drag. We want to offset based on the drag amount, so we'll recalculate the `currentPageOffset` as the sum of the current page position and the drag amount, always coerced between the maximum and minimum limits imposed by the page size and the amount of pages on each side.

```kotlin
with(state) {
    val pos = pageSize * currentPageOffset
    val max = if (currentPage == minPage) 0 else pageSize * offscreenLimit
    val min = if (currentPage == maxPage) 0 else -pageSize * offscreenLimit
    val newPos = (pos + dy).coerceIn(min.toFloat(), max.toFloat())
    currentPageOffset = newPos / pageSize
  }
```


### Measuring and placing children

```kotlin
Layout(
    children = { /* ... */ },
    modifier = modifier = { /* ... */ }
  ) { measurables, constraints ->
    layout(constraints.maxWidth, constraints.maxHeight) {
      val currentPage = state.currentPage
      val offset = state.currentPageOffset
      val childConstraints = constraints.copy(minWidth = 0, minHeight = 0)

      measurables
        .map {
          it.measure(childConstraints) to it.page
        }
        .forEach { (placeable, page) ->
          val xCenterOffset = (constraints.maxWidth - placeable.width) / 2
          val yCenterOffset = (constraints.maxHeight - placeable.height) / 2

          if (currentPage == page) {
            pageSize = placeable.width
          }

          val xItemOffset = ((page + offset - currentPage) * placeable.width).roundToInt()

          placeable.place(
            x = xCenterOffset + xItemOffset,
            y = yCenterOffset
          )
        }
    }
  }
```

Step by step:

* When creating a custom layout, you get some `constraints` imposed by the parent in the `MeasureBlock`. Here, those are used to impose the maximum possible width and height to the custom layout (see the `layout()` function call.
* Constraints are used to measure layouts, so it creates a copy of the parent constraints but with minimum `width` and `height` of 0. Those new constraints are used to measure each children using the `Measurable#measure` function, which returns a `Placeable` for that child. (More details on Constraints [here](https://developer.android.com/reference/kotlin/androidx/compose/ui/unit/Constraints)).
* Time to place the calculated placeables. For each child it calcualtes the center offset for both x and y to place the child centered to the available space. It also uses the chance to initialize the `pageSize` we got pending, now that we finally got its real measures. There is an extra offset for the item on X axis imposed by the `currentPageOffset`. For placing each child on the required `x` and `y` coordinates, it calls the `Placeable#place(x, y)` function for each generated `Placeable`.

And that's the end of the story.

<img width="200" src="assets/images/pager_drag.gif"/> 

### 📝 Final thoughts

This exercise of taking some sample code and going over it in a stepped way can be very rewarding as a learning process. I definitely recommend doing that from time to time 🙂

Remember this code is **not meant to be a final version** of the `Pager` composable just part of one of the Jetpack Compose samples. Google can evolve it in different ways and use it as trial and error to test and try components that could become part of the compose library at some point in the future. Not meaning this will be necessarily the case for this one. Time will tell.

The [official Jetpack Compose samples](https://github.com/android/compose-samples/) is a really good way to get familiarized with Compose codebases and architecture. I recommend you to look into it.

---

You might be interested in other posts I wrote about Jetpack Compose:

* [Awaiting next animation frame using suspend](https://jorgecastillo.dev/jetpack-compose-await-next-frame)
* [Sneak peek into Compose ConstraintLayout](https://jorgecastillo.dev/jetpack-compose-constraintlayout)
* [Custom layouts, measuring and WithConstraints Composable](https://jorgecastillo.dev/jetpack-compose-withconstraints)

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Jetpack Compose posts 🙌
