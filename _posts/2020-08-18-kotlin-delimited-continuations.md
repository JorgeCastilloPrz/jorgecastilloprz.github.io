---
layout: post
current: post
cover: assets/images/kyoto.jpeg
navigation: True
title: Delimited continuations in Kotlin
date: 2020-08-15 11:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Continuations represent the rest of a program. Delimited continuations capture just some part of it. 

### ⏩ Continuation

It is the implicit parameter that the Kotlin compiler passes to any `suspend` function when desugarizing. It is represented by an `interface`:

```kotlin
interface Continuation<in T> {
  abstract val context: CoroutineContext
  abstract fun resumeWith(result: Result<T>)
}
```

As you can see, it's like a plain callback. You can also find it [within the official docs](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/). 

A `Continuation` is **always linked to a coroutine**. Both terms are sometimes used indifferently because of that, but there are important differences.

A coroutine has a `Continuation` associated, which contains the state of the coroutine by providing a couple of things:

* A `CoroutineContext` used to run it. It tells the coroutine **how to suspend** itself.
* A callback to wire results in the end. It uses the [Kotlin Result datatype](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) for that purpose, so you can resume your program either with a `success` or a `failure` as a result. Following docs: 

> "It will resume the execution of the corresponding coroutine by passing the given value as the return value of the last suspension point."

This means the continuation represents the program flow. It decides how the program continues, and that makes the idea powerful to encode advanced control flow on top of it.

There are also a couple of extension functions as shortcuts:

* `fun <T> Continuation<T>.resume(value: T)`
* `fun <T> Continuation<T>.resumeWithException(exception: Throwable)`

And a constructor function [in the standard library](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) to create a Coroutine given a context and a callback function:

```kotlin
inline fun <T> Continuation(
    context: CoroutineContext,
    crossinline resumeWith: (Result<T>) -> Unit
): Continuation<T>
```

### 🤷‍♂️ What does it represent?

As explained above, a `Continuation` decides **how your program continues**, so it literally represents the program execution flow. It also carries information about the program state.

Let's briefly recap about Coroutines to understand this. 

Kotlin can `suspend` some work at a given point by starting a coroutine. That is be the afforementioned "suspension point". Here, you "suspend" your work and create a coroutine to perform some other work. Once there's a result of it, you can "resume" your work where you left it.

The `Continuation` is the way Kotlin remembers where the suspension points are and therefore where to return to. It also provides means to access information about **the state of the program right before the suspension point**. So it's literally a door to return the flow and the state of the program to the latest suspension point.

### 💡 How it is used

Let's say we've got a `suspend` function to perform some job we need to suspend, and a caller function like `main` in this case:

```kotlin
suspend fun doSomething() = "Done!"

suspend fun main() {
  doSomething()
}
```

If we remove the boilerplate and metadata to stay simple, that gets decompiled to something like this in Java:

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

   // $FF: synthetic method
   public static void main(String[] var0) {
      RunSuspendKt.runSuspend(new FileKt$$$main(var0));
   }
}
```

Obviating the need for an enclosing class given the lack of "package level" variables and functions in Java, the key point here is how both `suspend` functions have been converted to static functions that get the `Continuation` passed as an explicit argument, so it is used as a standard callback.

You can see how the `main` function needs to forward the continuation to the `doSomething($completion)` call.

All that is hidden to us by the Kotlin compiler, that allows to call `suspend` functions synchronously, while everything stays asynchronous under the hood. That's the magic.

### 🕵️‍♀️ Delimited Continuations

Exceptions in Kotlin can be understood as one way of control flow:

```kotlin
suspend fun loadSpeakers(): Either<String, List<Speaker>> =
  try {
    val speakers = fetchSpeakersFromNetwork()
    speakers.toDomain().right()
  } catch (e: Exception) {
    "Oops, not really done".left()
  }
```

Here, if `fetchSpeakersFromNetwork()` throws, the flow would jump to the `catch` block, meaning, you can understand `try / catch` as a **control flow operator**. It's like a GOTO.

This control operator also allows to pass explicit information for the jump. It's as simple as adding a parameter to your exception:

```kotlin
data class RichException(val data: Int) : Exception()

suspend fun doSomething(): Int =
  try {
    throw RichException(6)
    10
} catch (e: RichException) {
    e.data + 10
}
```

The result will be `16`, since we jump to the `catch` block and pass the value `6` to it.

[This example provided by Sebastian Fischer](https://gist.github.com/sebfisch/2235780) showcases how we could write this in a more generic way, if our language supported some specific control flow operators.

Let's translate this examples to Kotlin-like pseudocode. Let's say we have `capture` and `escape` control operators, so `capture` would return the result of the lambda block (`A` in this case), and `escape` is able to **escape the enclosing scope** and return another result instead (`B` here).

```kotlin
capture { A }
escape { continuation -> B }
```

Note that `escape` is also able to continue the computation of `capture` by using the `continuation` parameter. This `continuation` represents **the parameters passed to `escape`**.

Using these we could also represent the exception system, similar to `try/catch` but controlling execution flow in a more imperative way. Let's see a couple of examples:

```kotlin
capture { 2 * escape { _ -> 42 } }
```

Here, `escape` has the power to escape the `capture` closure and return `42` as the overall expression result instead.

We even have the power to continue the original (enclosing) computation by using the `continuation` parameter:

```kotlin
capture { 2 * escape { continuation -> 17 + continuation * 4 } }
```

We escape the capture closure by adding `17` to the `continuation`, which as explained above, captures **the parameter passed to escape** (`2` in this case). So the final result will be `17 + 2 * 4 = 25`.

In other words, we got the `capture { 2 * x }` computation that we `escape` with the result of the `scape` block, `17 + 2 * 4 = 25`.

We could use `capture` and `escape` to encode exception handling with something like:

```kotlin
throw e = escape { _ -> { handler -> handler(e) } }
try { a() } catch(e) { h(e) } = capture { 
  a().let { { it } }
}
```

* Throw can escape any enclosing computation by providing a new function that takes an exception handler that applies to the thrown exception. The `continuation` is ignored.
* Try / catch uses `capture` to evaluate `a()`, and in case `throw` is not called inside of it, it'll return the result o `a()`. If `throw` gets called, it'll escape the `try / catch` computation and return its lambda to apply the exception handler.


Delimited continuations are also used for control flow.

### 💡 Extra bullets

TBD (add relevant links and explanations)

---

You might be interested in other posts I wrote about Kotlin, like:

* [Delimited continuations gist by Sebastian Fischer](https://gist.github.com/sebfisch/2235780)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Kotlin posts 🙌
