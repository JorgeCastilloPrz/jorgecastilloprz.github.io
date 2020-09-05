---
layout: post
current: post
cover: assets/images/kyoto.jpeg
navigation: True
title: Kotlin Continuations
date: 2020-09-03 11:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Continuations represent the rest of a program. They are a form of control flow.

### 🤖 Compile time

Each time Kotlin finds a `suspend` function, that represents a **suspension point** that the compiler will desugarize into a callback style. For example:

```kotlin
suspend fun doSomething() = "Done!"

suspend fun main() { doSomething() }
```

If we remove the boilerplate and metadata to stay simple, that gets decompiled to something like this in Java (using the `tools` -> `Kotlin` -> `Show Kotlin Bytecode` IntellIJ menu option):

```java
// FileKt.java
public final class FileKt {
   @Nullable
   public static final Object doSomething(@NotNull Continuation $completion) {
      return "Done!";
   }

   @Nullable
   public static final Object main(@NotNull Continuation $completion) {
      Object var10000 = doSomething($completion);
      return var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED() ? var10000 : Unit.INSTANCE;
   }
   
   // ...
}
```

Obviating the need for an enclosing class given the lack of "package level" functions in Java, the key point here is how both `suspend` functions have been converted into **static functions** that get the `Continuation` passed as an explicit argument. This is formally called CPS (*[Continuation Passing Style](https://en.wikipedia.org/wiki/Continuation-passing_style)*).

You can see how the `main` function needs to forward the `$completion` continuation to the `doSomething()` call.

All that is hidden to us by the Kotlin compiler, that allows to call `suspend` functions synchronously, while everything stays asynchronous under the hood. That's the magic and the point of all this.

### ⏩ Continuation

So, as explained above, every `Continuation` is **associated with a suspension point**. A continuation is the implicit parameter that the Kotlin compiler passes to any `suspend` function when compiling it. It is represented by a basic contract:

```kotlin
interface Continuation<in T> {
  abstract val context: CoroutineContext
  abstract fun resumeWith(result: Result<T>)
}
```

Under the hood, Kotlin will generate a `ContinuationImpl` for this contract for each suspended function, which we will dive into in further sections on this post.

As you can see, it looks like a callback, because that is essentially what it is. You can also find it [in the official docs](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/). It's composed of:

* A `CoroutineContext` that tells the coroutine **how to suspend** itself.
* A callback to wire results in the end. It uses the [Kotlin Result datatype](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) for that purpose, so you can resume your program either with a `success` or a `failure` (exception) as a result. Following docs:

> "It will resume the execution of the corresponding coroutine by passing the given value as the return value of the last suspension point."

So, the continuation decides how the program continues after some work, and that makes it **another form of control flow**. It will be used to coordinate the work between all our `suspend` functions.

There are also a couple of extension functions as shortcuts, in case you need to deal with continuations manually at some point:

* [`fun <T> Continuation<T>.resume(value: T)`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume.html)
* [`fun <T> Continuation<T>.resumeWithException(exception: Throwable)`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume-with-exception.html)

And a constructor function [in the standard library](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) to create a Continuation given a context and a callback function:

```kotlin
inline fun <T> Continuation(
    context: CoroutineContext,
    crossinline resumeWith: (Result<T>) -> Unit
): Continuation<T>
```

This is handy to have, but overall in Kotlin **you'll likely not need to write your own Continuations**, unless you are building your own libraries like we do with [Arrow](https://arrow-kt.io/), and only for some very specific use cases. There's a little bit more on this topic below.

You will likely rely on the Kotlin `suspend` support instead, and let the Kotlin compiler desugar it under the hood for you.

### ⚙️ How does it work internally?

Recapping about `suspend` functions:

> A `suspend` function is a function that can be suspended (paused) and resumed later on. They can execute long running operations in a non blocking way.

Keeping that in mind, Kotlin can suspend some work at arbitrary points within a Coroutine or a `suspend` function, as soon as it finds any `suspend` function calls. Those would be the afforementioned **suspension points**.

Like `program`, `firstOp` and `secondOp` in this example:

```kotlin
suspend fun firstOp() = true
suspend fun secondOp(firstRes: Boolean) = 1

suspend fun program(): Int {
  val firstRes = firstOp()
  return secondOp(firstRes)
}
```

From the Android docs:

> Kotlin uses a stack frame to manage which function is running along with any local variables. When suspending a coroutine, the current stack frame is copied and saved for later. When resuming, the stack frame is copied back from where it was saved, and the function starts running again.

That is how the flow and the state of the program can be stored and lately resumed for each suspension point.

For coordinating the different `suspend` functions, Kotlin generates a **finite state machine**. It creates a label for each suspension point, so it has a way to jump from one to another. Then it's able to call our suspend function recursively with different label values to jump between the different states as required. 

The `Continuation` will keep track of the current label at any point in time, and will be updated accordingly.

For each suspension point reached, the `Continuation` is passed, and lately updated with the corresponding result of the computation (so the next one can have access to it) and the corresponding next label value for the following recursion.

All this is handled into a conveniently generated `ContinuationImpl` instance that implements the `Continuation` contract and is passed around the different states.

I'm intentionally staying a bit high level here, because there is [this awesome post](https://proandroiddev.com/how-do-coroutines-work-under-the-hood-803e6e9da8bb) by Ashish Kumar for in depth details 👌

About storing and restoring state before and after suspension points, there are also some interesting gotchas on [this post](https://medium.com/@lucianoalmeida1/an-overview-on-kotlin-coroutines-d55e123e137b) by Luciano Almeida 🙏

### 🕵️‍♀️ Java interoperability

As you probably imagine already, the **Kotlin compiler plays a big role** for making the `suspend` magic work. That means you will get the most out of this feature if you call suspended functions from Kotlin, not meaning they are not supported at all from Java.

Since our `suspend` functions are translated into **static** functions within a class, we should be able to call them from java using a non sugarized standard callback style, by passing an explicit instance of a `Continuation`:

```java
public class Main {

  public static void main(String[] args) {
    FileKt.doSomething(new Continuation<String>() {
      @Override
      public void resumeWith(@NotNull Result<String> o) {
        // process result `o`
      }

      @NotNull
      @Override
      public CoroutineContext getContext() {
        return EmptyCoroutineContext.INSTANCE;
      }
    });
  }
}
```

One issue here is that starting on Kotlin 1.3, `Continuation` uses the [Kotlin Result](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) inline class for the `resumeWith` method, and **inline classes are not supported by Java**. If you want to use them you must provide explicit wrappers from Kotlin. More details on [this issue](https://discuss.kotlinlang.org/t/inline-classes-tedious-to-use-considering-java-interop/9382).

So, if you try to compile the above Java snippet with the anonymous `Continuation` implementation, it will not compile because of the `Result` type.

One thing you could do to overcome this issue would be to provide your own `Continuation` implementation that maps the Kotlin result to something else, to get rid of it and open the usage from Java. We actually [do that in the Arrow library](https://github.com/arrow-kt/arrow-core/blob/master/arrow-core-data/src/main/kotlin/arrow/typeclasses/internal/Continuation.kt), but with a different ultimate purpose.

But if you really want to call your `suspend` functions from Java because you might be facing a gradual migration, there are simpler ways.

One approach would be to translate the `suspend` call to a JDK 8 `CompletableFuture`, so the bridging allows you to keep the asynchronous use case covered:

```kotlin
fun doSomethingFromJava() =
  GlobalScope.future { doSomething() }

// Or with explicit CoroutineContext
fun doSomethingFromJava() =
  GlobalScope.future(Dispatchers.IO) { doSomething() }
```

That will require the jdk8 coroutines integration module though:
```groovy
dependencies {
  implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-jdk8'
}
```

And you'll be able to call it from Java:

```java
CompletableFuture<String> future = FileKt.doSomethingFromJava();
String result = future.get();
```

This is possible because the bridging is done on the Kotlin side, hence the `Result` type is not visible from Java anymore.

> Note that here we're using `GlobalScope` for the sake of the example, but you should likely use a narrower `CoroutineScope` that fits your requirements in case you are using KotlinX Coroutines library. Since that logic is on the Kotlin side, you're free to apply any design you want in that sense.

Another alternative would be to **use the KotlinX Coroutines builders from Java**:

```kotlin
Deferred<String> deferred = BuildersKt.async(
  GlobalScope.INSTANCE,
  Dispatchers.getIO(),
  CoroutineStart.DEFAULT, // CoroutineStart.LAZY, or other strategies
  (Function2<CoroutineScope, Continuation<? super String>, String>) (coroutineScope, continuation) -> {
    // do your stuff
    return "Some result";
  }
);

Job job = BuildersKt.launch(
  GlobalScope.INSTANCE,
  Dispatchers.getIO(),
  CoroutineStart.DEFAULT, // CoroutineStart.LAZY, or other strategies
  (Function2<CoroutineScope, Continuation<? super Unit>, Unit>) (coroutineScope, continuation) -> {
    // do your stuff
    return Unit.INSTANCE;
  }
);

try {
  // Usually used to bridge regular blocking code to libraries using suspend, 
  // to be used in main functions like from tests or similar.
  String result = BuildersKt.runBlocking(
    Dispatchers.getIO(),
    (Function2<CoroutineScope, Continuation<? super String>, String>) (coroutineScope, continuation) -> {
      // do your stuff
      return "Some result";
    }
  );
} catch (InterruptedException e) {
  // If this blocked thread is interrupted, then the coroutine job is cancelled and
  // * this runBlocking invocation throws InterruptedException.
  // *
  // Do something with the interruption error
}
```

These are the standard KotlinX coroutines launchers, but called from Java, so it obviously doesn't look that idiomatic. Just another interesting option to share.

Finally, you can make it even easier, and **simply provide Kotlin functions that launch the required coroutines and call those from Java**.

### 😯 Explicit usages of the Continuation in Kotlin

As we said you'll likely not use continuations explicitly in Kotlin, except when you do. There are some use cases where you want control over when the continuation is used to resume the state of your program on a given suspension point. We can see this when we are wrapping callback style apis with `suspendCoroutine`:

```kotlin
suspend fun syncClick(): Unit = suspendCoroutine<Unit> { cont ->
  emptyStateIcon.setOnClickListener { 
    cont.resumeWith(Result.success(Unit)) 
  }
}
```

Here we get a `cont` parameter that's our `Continuation` that we can use to resume our program with the required result. These wrappers are often used in Android to wrap system listeners to capture user interaction, for example.

But this is dangerous usage actually, because coroutines are **not multishot**. Meaning second time you click on the button, it'll try to resume an already resumed coroutine using the same continuation, and therefore crash 💥. So you should always **detach your listener** after the first time:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
  // ...
  lifecycleScope.launchWhenStarted {
    val result = syncClick()
    result // "Clicked"
  }
}

suspend fun syncClick(): String = suspendCoroutine { cont ->
  emptyStateIcon.setOnClickListener {
    cont.resumeWith(Result.success("Clicked"))
    emptyStateIcon.setOnClickListener(null) // detach!
  }
}
```

This will ensure you get your `result` as "Clicked" only once. 

This is probably not the best example in the world, since an event like button clicks would be better represented as a `Stream` of events you can observe, not a single one, but it's a starting point to understand how this works.

You'll likely use this wrapping style to convert one time triggered async logics, like waiting for a View Layout, for example. Those one time calls that you proactively want to call and ensure they are done before stepping into something else. There's a good example of that in [this detailed post by Chris Banes](https://medium.com/androiddevelopers/suspending-over-views-19de9ebd7020).

This is also widely used to wrap things like network, database, file read requests or the like, where third party apis might impose a callback based style, so by wrapping them like this you can interoperate with them **synchronously instead**.

That said, it is **highly recommendable** to use the cancellable alternative instead: `suspendCancellableCoroutine`. 

It provides a [CancellableContinuation](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellable-continuation/index.html) instead we can use to wire up cancellation. We could use that in Android to release a listener if the caller coroutine gets cancelled in the scope, for example.

(I'll copy Chris Banes example here since it's pretty self explanatory 🙏)

```kotlin
suspend fun View.awaitNextLayout() = suspendCancellableCoroutine<Unit> { cont ->
    // This lambda is invoked immediately, allowing us to create
    // a callback/listener

    val listener = object : View.OnLayoutChangeListener {
        override fun onLayoutChange(...) {
            // The next layout has happened!
            // First remove the listener to not leak the coroutine
            view.removeOnLayoutChangeListener(this)
            // Finally resume the continuation, and
            // wake the coroutine up
            cont.resume(Unit)
        }
    }
    // If the coroutine is cancelled, remove the listener
    cont.invokeOnCancellation { removeOnLayoutChangeListener(listener) }
    // And finally add the listener to view
    addOnLayoutChangeListener(listener)

    // The coroutine will now be suspended. It will only be resumed
    // when calling cont.resume() in the listener above
}
```

Then you can use it like:

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
  // do some stuff...
  
  // Wait for the next layout pass to know height of the view
  titleView.awaitNextLayout()

  // Do some other stuff that depends on the view measures...
}
```

Here, he makes sure that the listener is detached whenever the parent coroutine gets cancelled. In the other hand, if the asynchronous api you are wrapping provides cancellation capabilities, you might want to perform **bidirectional cancellation** and cancel the parent coroutine in return, also. There is also an example of that [in the mentioned blogpost](https://medium.com/androiddevelopers/suspending-over-views-19de9ebd7020).

> Note that the cancellable variant [is provided by KotlinX Coroutines](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html), not the standard library. **Kotlin stdlib doesn't provide cancellation support**, that is provided by runtime libraries built on top of it like [KotlinX Coroutines](https://github.com/Kotlin/kotlinx.coroutines) or [Arrow Fx Coroutines](https://github.com/arrow-kt/arrow-fx/tree/master/arrow-fx-coroutines). Both libraries provide support for collaborative cancellation.

For cases like user interactions you'd likely prefer a Stream based solution, so for wrapping those you can use either `callbackFlow {}` [from KotlinX Coroutines](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html) or `Stream.cancellable {}` [from Arrow Fx Coroutines](https://github.com/arrow-kt/arrow-fx/blob/fb9c1fc18d2d956fb0fa3da3ff443513f73da6c9/arrow-fx-coroutines/src/test/kotlin/arrow/fx/coroutines/stream/CallbackTest.kt#L47).

Here is an example of how it would look like using the **Functional Streams** implementation by the upcoming Arrow Fx Coroutines, only as a sneak peek.

```kotlin
fun SwipeRefreshLayout.refreshes(): Stream<Unit> = Stream.cancellable {
  val listener = SwipeRefreshLayout.OnRefreshListener {
    emit(Unit)
  }
  this@refreshes.setOnRefreshListener(listener)
  CancelToken { this@refreshes.setOnRefreshListener(null) }
}
```

I'm not getting deeper into this topic on this post since it's out of scope, but you can expect some posts about this idea in the future 👍

### 💡 Extra bullets

Continuations or Continuation Passing Style is a concept that you can find in Kotlin for encoding control flow, but it's essentially a generic programming concept not only tied to Kotlin. You could take it further and implement "delimited continuations" based on a similar but a bit more advanced idea. When a continuation represents the rest of a program, a delimited continuations captures only some part of it. This is a wide topic with a lot of papers published since quite long ago that could start shining soon in Kotlin 😉

Here you have some links of interest:

* [This gist](https://gist.github.com/sebfisch/2235780) by [Sebastian Fischer](https://gist.github.com/sebfisch) is an interesting source for starting to dig into the concept of Delimited Continuations.
* [This post](https://cs.ru.nl/~dfrumin/notes/delim.html) by [Dan Frumin](https://cs.ru.nl/~dfrumin/).
* [Suspending over Views](https://medium.com/androiddevelopers/suspending-over-views-19de9ebd7020) by Chris Banes.
* [Suspending over Views - Example](https://medium.com/androiddevelopers/suspending-over-views-example-260ce3dc9100) by Chris Banes.
* [Suspend functions under the hood](https://proandroiddev.com/how-do-coroutines-work-under-the-hood-803e6e9da8bb) by Ashish Kumar.
* [Continuation Passing Style](https://en.wikipedia.org/wiki/Continuation-passing_style).
* [An Overview on Kotlin Coroutines](https://medium.com/@lucianoalmeida1/an-overview-on-kotlin-coroutines-d55e123e137b) by [Luciano Almeida](https://medium.com/@lucianoalmeida1).

---

You might be interested in other posts I wrote about Kotlin, like:

* [Kotlin SAM conversions](https://jorgecastillo.dev/kotlin-sam-conversions)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Kotlin posts 🙌
