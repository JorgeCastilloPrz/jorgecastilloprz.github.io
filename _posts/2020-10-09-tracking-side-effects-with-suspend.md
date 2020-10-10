---
layout: post
current: post
cover: assets/images/kyoto2.jpeg
navigation: True
title: Tracking side effects at compile time with suspend
date: 2020-10-10 10:00:00
tags: [kotlin, android]
class: post-template
subclass: 'post'
author: jorge
---

Thinking of suspend as a Kotlin stdlib mechanism for flagging and tracking effects at compile time.

### Going declarative

Functional Programming puts a lot of emphasis on achieving **concern separation** between the pure logics of a program (**algebras**) and the **runtime** used to run it.

To guarantee this separation, we need mechanisms that allow us to **represent our program in memory**, so we can later **interpret that program with a runtime**. This clear decoupling between the actual program description and the runtime, allows to swap runtime execution strategies, and even apply desired runtime optimizations.

One example of this is the [Kotlin stdlib's Sequences](https://kotlinlang.org/docs/reference/sequences.html). Those provide operators to work over the elements lazily while staying declarative. In other words, deferred.

Then we have a few **terminal operators**, which only exist for ultimately consuming / executing the in memory program created and ultimately apply optimizations.

From the docs:

> "Actual computing happens only when the result of the whole processing chain is requested."

```kotlin
val words = "The quick brown fox jumps over the lazy dog".split(" ")

val wordsSequence = words.asSequence() // convert list to a sequence

val lengthsSequence = wordsSequence.filter { println("filter: $it"); it.length > 3 }
    .map { println("length: ${it.length}"); it.length }
    .take(4)

println("Lengths of first 4 words longer than 3 chars")

// terminal operation: obtaining the result as a List
println(lengthsSequence.toList())
```

Try copying this code, commenting the final line and running it. You'll see how the sequence is never processed. Before calling terminal operators, what we have is only a **description of a program** waiting to be executed.

Another example of this can be found [in Jetpack Compose](https://developer.android.com/jetpack/compose/mental-model), in case you do Android.

Compose allows us to use `@Composable` functions as the atomic pieces to create an in memory description of our UI. Each function describes a **UI effect**. Then, there is a runtime prepared to interpret that **description tree** and apply desired optimizations.

Some examples of runtime optimizations in Compose could be running compositions in parallel, offloading compositions to different threads, running those in different order, or features like [smart recomposition](https://developer.android.com/jetpack/compose/mental-model#recomposition).

`@Composable` functions have restricted usage. The compose compiler will enforce us to call them from another composable function or an **environment** prepared to run UI effects 👉 <u>An integration point like `setContent {}`</u>.

> This is particularly important, since this restriction ensures we cannot call composable functions from anywhere, but always from an environment that is able to interpret them and apply the required optimizations.

But, can we also use these ideas for composing our program pure logics (algebras)? Of course we can, by using `suspend`.

> Suspend is the Kotlin stdlib mechanism to make effects visible for the compiler, so we enforce them to run under an environment that is prepared to run effects. A.K.A: Coroutine, or in other words, a prepared **runtime**.

[This Tweet](https://twitter.com/relizarov/status/1314116971286466560) by Roman Elizarov, team lead at JetBrains describes this idea pretty well.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">The suspend keyword in Kotlin colors parts of the code, marking it as asynchronous, enforcing constraints on how it can be used. This is a good thing. We should not do network I/O calls from anywhere in our code, but only from specially designated places in our architecture.</p>&mdash; Roman Elizarov (@relizarov) <a href="https://twitter.com/relizarov/status/1314116971286466560?ref_src=twsrc%5Etfw">October 8, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

We shouldn't make potentially harmful calls from anywhere, but from an environment that is prepared for it.

### Making effects pure

We learned the benefits of concern separation, and that we can achieve it using the power of `suspend`.

> Suspend is the Kotlin stdlib mechanism for flagging effects 👉 ensure they're called from the correct places.

To make an effect pure, just flag it as `suspend`.  By doing this we'll be making Kotlin compiler aware of it, so we'll not be able to call the effect from anywhere. This will shorten the feedback loop by **moving side effect tracking to compile time**. If we try to call an effect from a not allowed place, we'll get an instant red underline.

Let's go ahead and flag all effects as `suspend` in our architecture. Those will likely be closer to the architecture edges, like those places where we load data from network, caches, persistence, access file system, display something on screen, etc.

```kotlin
interface UserService { // Retrofit service
  suspend fun loadUser(): User = TODO()
}

class UserPersistence {
  suspend fun loadUser(): User = TODO()
}

class AnalyticsTracker {
  suspend fun trackEvent(event: Event): Unit = TODO()
}
```

This action will enforce us to also flag the callers as `suspend`, and this requirement will crawl up our call stack to effectively enforce us to convert our complete program in a description of a program, meaning going declarative. ✅

At some point we will reach the integration point, where we'll require to provide a runtime to run our suspended program 👉<u>a Coroutine</u>. This will ideally happen as early as possible in our architecture, or in other words, be **as close as possible to our program's entry point**.

This is because we are looking for writing our complete program in this declarative style, hence making it completely **pure**. Our program logics as the algebras that describe our program.

Then we'll leave the runtime as a very thin layer that works as an integration point intentionally constrained to the entry point.

### The runtime

We already know how to achieve concern separation by suspending effects, and our ultimate goal of achieving purity for the complete program. That will also make it more testable, since **purity means determinism, and therefore predictability**.

To provide a decoupled runtime, and for the case of programs with suspended effects (also known as effectful programs), we will need to create a coroutine.

In case we are working with object oriented / imperative style, we are probably using KotlinX Coroutines. We can use the coroutine builders to create one, like [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) and [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html), which will require a `CoroutineScope` for the structure concurrency:

```kotlin
fun main() {
  coroutineScope {
    launch {
       // Our suspended program 🍃
    }
  }
}
```

This is just an intentionally simple example, but we'll need to be conscious about the convenient scope to use per use case, or how to create it. The key point here is that we've created an environment for running a suspended program.

Let's say we're not using KotlinX Coroutines, but only the stdlib coroutines support (`suspend`, [`CoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/), [`startCoroutine`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html), [`suspendCoroutine`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html)... etc), we can also have our environment (Coroutine) created like:

```kotlin
suspend fun program(): Unit = TODO("Our suspended program 🍃")

fun main() {
  ::program.startCoroutine(
    Continuation(
      context = EmptyCoroutineContext,
      resumeWith = { result -> /* Kotlin Result */ }
    ))
}
```

So we can create a coroutine over our suspended program with the [`startCoroutine` stdlib call](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html). It's a bit more convoluted, since we'll have to provide implementations for the [`CoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/), and the [`Continuation`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html).

Truth is we'll probably work like this if writing a library built on top of the Kotlin stdlib suspend support. In that case we'd probably not want to pack KotlinX Coroutines into our library, since it's a quite high level library, but work over the stdlib suspend machinery instead to stay efficient. After all, that is also how KotlinX Coroutines was built.

Moving into the Functional Programming world, we've got the Arrow Fx Coroutines library instead, which provides the [`Environment`](https://github.com/arrow-kt/arrow-fx/blob/b342185cef8e33fc126715b4b58f7ee3ab7d8bd9/arrow-fx-coroutines/src/main/kotlin/arrow/fx/coroutines/Enviroment.kt#L21):

```kotlin
fun main() {
  val env = Environment()
  val cancellable = env.unsafeRunAsyncCancellable(
    { // Our suspended program 🍃 },
    { e -> /* handle errors unhandled by the suspended program */ },
    { a -> /* handle result of the program */ }
  )
}
```

The `Environment` also provides a runtime and takes care of the execution strategy used, providing methods to run our program [synchronously](https://github.com/arrow-kt/arrow-fx/blob/b342185cef8e33fc126715b4b58f7ee3ab7d8bd9/arrow-fx-coroutines/src/main/kotlin/arrow/fx/coroutines/Enviroment.kt#L40), [asynchronously](https://github.com/arrow-kt/arrow-fx/blob/b342185cef8e33fc126715b4b58f7ee3ab7d8bd9/arrow-fx-coroutines/src/main/kotlin/arrow/fx/coroutines/Enviroment.kt#L58), or [asynchronously and cancellable](https://github.com/arrow-kt/arrow-fx/blob/b342185cef8e33fc126715b4b58f7ee3ab7d8bd9/arrow-fx-coroutines/src/main/kotlin/arrow/fx/coroutines/Enviroment.kt#L67), so we can retain a cancellation token to invoke later on and **cancel the whole execution tree**.

We can also pass an explicit `CoroutineContext` for executing the program, which by default is the `ComputationPool`. The `Environment` is actually an interface, so we can provide **our own runtimes** in case we want to, and in case [the default one](https://github.com/arrow-kt/arrow-fx/blob/b342185cef8e33fc126715b4b58f7ee3ab7d8bd9/arrow-fx-coroutines/src/main/kotlin/arrow/fx/coroutines/Enviroment.kt#L86) provided by Arrow doesn't fit our needs.

In case we want to go for the functional approach, we'll get access to all the functional effect apis, functional data types, concurrency operators, functional Streams, out of the box cancellation, and all the functional goodies the library provides. All of them are meant to work seamlessly in combination with the `Environment`.

One example of this can be the advanced concurrency operators from Arrow Fx Coroutines. Those are meant to run effects, so they're also tagged as `suspend`, meaning we can't really run those outside of a controlled environment. You can have a sneak peek on those [on the slides for my Android Summit talk](https://www.47deg.com/presentations/2020/10/08/functional-android-apps/).

For runtime optimizations, the Arrow team is preparing more advanced work at the `Continuation` level that will enable quite interesting behaviors that the Kotlin stdlib doesn't support at this point. That will be unveiled in the future, but will leverage the capabilities of concern separation even more. Stay tunned for updates 🙌

### Suspended entry points

Some platforms / frameworks provide suspended entry points, like the well known Kotlin `suspend main` function, or the Ktor support for writing suspended endpoints.

If a framework provides this, that means the platform is providing us a preconfigured environment to run suspended programs. Therefore, we'll likely not need to provide one, unless we really want / need to replace it.

Most of the time the platform will have appropiately configured / optimized the environment for the average platform use case. That is picking the proper coroutine context, for example. But in case we want to leverage the upcoming capabilities of Arrow in terms of runtime optimization, we'll need to replace it with the Arrow `Environment`.

We'll likely write more in depth literature about all that when the time comes.

### What about writing our pure program?

Once we've got a runtime to interpret our in memory program, and a means to flag side effects to keep those under control by the compiler, we'll start writing our pure declarative program logics.

In the case of Arrow, we might want to use data types like `Either` or `Validated` and stay under the suspend umbrella, so we always need a proper environment to run the suspended effectful program. We'll also be able to leverage the `Either` computational blocks and the bindings for composing the program logics through a neat direct syntax.

I'm planning to write a detailed post on how to write pure logics for our decoupled architectures that will showcase all that plus a lot of concurrency operators, so I'll not dive deep into that just yet. For now you can have a look to the mentioned [Android Summit talk](https://www.47deg.com/presentations/2020/10/08/functional-android-apps/).

### 📝 Final thoughts

Any programs can be written in an **eager** or a declarative deferred way. If we go for eager apis our computations will run as we created them, removing any chance of representing the program in memory and interpreting it in different ways later on.

For going declarative we could use data types encoded to be intentionally deferred / lazy, or just `suspend`.

---

I want to thank [47 Degrees](https://www.47deg.com/) for giving me the chance to learn and communicate all this in diverse forms.

You might be interested in other posts I wrote about Kotlin:

* [Android Summit talk covering all described in this post in depth](https://www.47deg.com/presentations/2020/10/08/functional-android-apps/)
* [Kotlin Continuations](https://jorgecastillo.dev/digging-into-kotlin-continuations)
* [Kotlin SAM support](https://jorgecastillo.dev/kotlin-sam-conversions)
* [All Kotlin posts including the FP ones](https://jorgecastillo.dev/tag/kotlin/)

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

More interesting stuff to come 🙌
