---
layout: post
current: post
cover: assets/images/racoon.png
navigation: True
title: Await next frame in Jetpack Compose
date: 2020-05-17 11:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Learn why view invalidation per se is not a thing in Jetpack Compose.

### 🤷‍ Use case

In Android, when we are drawing something to `Canvas` we usually enforce `View` invalidation via mechanisms like `View#invalidate()`. That is to enforce the system to perform a new drawing pass on the `View` according to new imposed requirements, or to a new state. Then we can rely on elapsed time to calculate how to render the current animation tick.

[Here is a library](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders) I wrote ages ago that made use of that concept.

Ultimately, we can understand any animation as a succession of frames where each one renders according to a state for the current snapshot in time. In Compose these animations are also possible, but **they rely on the Kotlin suspend system**.

### ✅ Jetpack Compose solution

I'll show the solution [originally shared by @adamwp](https://twitter.com/adamwp/status/1269653761980391425?s=20) first then we'll discuss a bit about the implementation.

```kotlin
@Composable
fun animationTimeMillis(): State<Long> {
  val millisState = state { 0L }
  val lifecycleOwner = LifecycleOwnerAmbient.current
  launchInComposition {
    val startTime = withFrameMillis { it }
    lifecycleOwner.whenStarted {
      while (true) {
        withFrameMillis { frameTime ->
          millisState.value = frameTime - startTime
        }
      }
    }
  }
  return millisState
}
```

This snippet uses the latest developer preview available for Jetpack Compose, which [can be found here](https://developer.android.com/jetpack/androidx/releases/compose). It's `0.1.0-dev15` as of today.

Focusing on the proposed solution, we could highlight the following ideas:

* It's a **composable** function that returns an immutable `State<Long>` we can observe from any composable, so it recomposes every time the state gets updated. This state will reflect the **elapsed animation time** every frame.
* The state is a `Long` value reflecting an elapsed amount of time in milliseconds. It is initialized as `0L`, and grows linearly from there.
* `launchInComposition` block launches a `suspend` side effect right when the composition is called, and that also **gets cancelled** as soon as this composable leaves the composition to avoid leaks. [Per the official docs](https://developer.android.com/reference/kotlin/androidx/compose/package-summary#launchincomposition):

>  The lambda *"will run in the `apply` scope of the composition's `Recomposer`, which is usually your UI's main thread"*.

> "Recomposition does not cause this block to be called again. To do that, there are [some launchInComposition overloads](https://developer.android.com/reference/kotlin/androidx/compose/package-summary#launchincomposition_1) that accept keys as additional arguments. Whenever those keys change, previous recomposition gets cancelled and the block runs again".

So we are scoping our task to the composition.

* First thing we do inside the block is to record the starting time when the animation starts. That is used to calculate the elapsed time every frame. We use `withFrameMillis { it }` which basically **supends until a new frame is requested**. You can find more details [here](https://developer.android.com/reference/kotlin/androidx/compose/dispatch/package-summary#withframemillis).

> Compose also provides [a withFrameNanos variant](https://developer.android.com/reference/kotlin/androidx/compose/dispatch/package-summary#withframenanos) that spits nanoseconds instead. These times rely on a [`MonotonicFrameClock`](https://developer.android.com/reference/kotlin/androidx/compose/dispatch/MonotonicFrameClock).

* We want to start our animation as soon as the enclosing `LifecycleOwner` gets **started**, not before. At that point we start an infinite loop that will `suspend` to wait for the next frame every time, and update the state with the currently elapsed time. This will represent our "draw invalidation", since observing composables will recompose each time.
* We can use `LifecycleOwnerAmbient.current` to retrieve the current enclosing `LifecycleOwner`. Remember `Ambients` are a mechanism by Compose that rely on Providers to be implicitly "injected" down the tree. They are usually used as a means to provide access to things required by many levels down the tree, so they don't need to be manually passed. You can find a detailed explanation of this [here](https://developer.android.com/reference/kotlin/androidx/compose/Ambient).
* Last thing we need to do is observe the returned state from any other `@Composable`, and be are ready to go 🥳

### 😯 Example

[ComposeFillableLoaders](https://github.com/JorgeCastilloPrz/ComposeFillableLoaders) sample can work as a good code sample that makes use of this idea to create a complex animation based on `Paths` drawn to the `Canvas`. The library [adds a new variant to the animation state](https://github.com/JorgeCastilloPrz/ComposeFillableLoaders/blob/f2abe60435dc7f1577d8bc69f79efa03f713987d/app/src/main/java/dev/jorgecastillo/fillableloader/FillableLoader.kt#L88), to indicate which animation phase it is in at the current time snapshot.

<img width="300" src="https://raw.githubusercontent.com/JorgeCastilloPrz/ComposeFillableLoaders/master/assets/watercat.gif"/>

---

This post was written using **Android Studio 4.2 Canary 4**. Remember you need the latest canary to run the latest version of Jetpack Compose.

You might be interested in other posts I wrote about Jetpack Compose, like:

* [Jetpack Compose ConstraintLayout](https://jorgecastillo.dev/jetpack-compose-constraintlayout)
* [Jetpack Compose WithConstraints](https://jorgecastillo.dev/jetpack-compose-withconstraints)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
