---
layout: post
current: post
cover: assets/images/yak.png
navigation: True
title: Suspend to await next frame in Jetpack Compose
date: 2020-05-17 11:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Learn why view invalidation is not needed in Jetpack Compose.

### 🤷‍ Use case

When we are drawing something to `Canvas` in Android we are used to enforce `View` invalidation with mechanisms like `View#invalidate()`. That is to enforce the system to render the `View` again according to new requirements imposed, or to a new state.

We've used that many times in the past to encode animations using the `Canvas`. [Here is a library](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders) I wrote ages ago that made use of that concept. Ultimately, we can understand any animation a succession of frames that render according to a state for the current snapshot in time.

In Jetpack Compose, these animations are also possible, but **they don't require explicit invalidation**.

### ✅ Proposed solution by Jetpack Compose

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

As you can see, some method signatures have changed in later Compose developer previews. We'll get to those in a second. Latest developer preview for Jetpack Compose [can be found here](https://developer.android.com/jetpack/androidx/releases/compose), and it's `0.1.0-dev15` as of today.

Focusing on the proposed solution, we should highlight the following:

* The function is essentially a **composable** function that returns an immutable `State<Long>`. We can observe this state from any composable, make it recompose every time the state gets updated.
* The state will be a `Long` reflecting an elapsed amount of time in milliseconds. It is initialized as `0L`, and grows linearly from there.
* `launchInComposition` block ensures we launch a `suspend` side effect right when the composition gets called, and that **it gets cancelled** as soon as this composable leaves the composition. As official documentation states, the lambda passed to it ["will run in the `apply` scope of the composition's `Recomposer`, which is usually your UI's main thread"](https://developer.android.com/reference/kotlin/androidx/compose/package-summary#launchincomposition). Recomposition does not cause this block to be called again. To do that, there are [some launchInComposition overloads](https://developer.android.com/reference/kotlin/androidx/compose/package-summary#launchincomposition_1) that allow us to pass keys as additional arguments. Whenever those passed keys change, previous recomposition gets cancelled and the block runs again.
* First thing we do inside `launchInComposition` is to record the starting time when the animation started. That will be used to calculate the elapsed time every frame. We use `withFrameMillis { it }` which basically **supends until a new frame is requested**. You can find more details [here](https://developer.android.com/reference/kotlin/androidx/compose/dispatch/package-summary#withframemillis). There is also [a `withFrameNanos` variant](https://developer.android.com/reference/kotlin/androidx/compose/dispatch/package-summary#withframenanos) that spits out nanoseconds instead.
* Next thing we do is scoping our task to the `LifecycleOwner` lifecycle. So we start our animation as soon as the owner is started, and not before. At that point we start an infinite loop that will wait for the next frame every time, and update the state with the currently elapsed time. That'll essentially emit a new elapsed time value on every frame.
* Last thing we need to do is observe the returned state from any other `@Composable`. And we are ready to go 🥳

### 😯 Example

[ComposeFillableLoaders](https://github.com/JorgeCastilloPrz/ComposeFillableLoaders) library can work as a good code sample that makes use of this idea to create a complex animation based on `Paths` drawn to the `Canvas`. The library [adds a new variant to the animation state](https://github.com/JorgeCastilloPrz/ComposeFillableLoaders/blob/f2abe60435dc7f1577d8bc69f79efa03f713987d/app/src/main/java/dev/jorgecastillo/fillableloader/FillableLoader.kt#L88), to indicate which animation phase it is in at the current time snapshot.

---

This post was written using **Android Studio 4.2 Canary 4**. Remember you need the latest canary to run the latest version of Jetpack Compose.

You might be interested in other posts I wrote about Jetpack Compose, like:

* [Jetpack Compose ConstraintLayout](https://jorgecastillo.dev/jetpack-compose-constraintlayout)
* [Jetpack Compose WithConstraints](https://jorgecastillo.dev/jetpack-compose-withconstraints)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
