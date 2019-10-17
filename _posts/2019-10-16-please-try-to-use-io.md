---
layout: post
current: post
cover: assets/images/murcia.jpg
navigation: True
title: Please Try to use IO!
date: 2019-10-16 12:36:00
tags: [kotlin, fp]
class: post-template
subclass: 'post'
author: jorge
---

A beautiful pic of Murcia, my beloved hometown 💛. Let's dive into why we consider `IO` safer than `Try` and why we recommend using it.

### How people is using Try

We've observed that many Kotlin devs have been using `Try` to control effects, and particularly for capturing exceptions thrown by effectful third party libraries (e.g: network requests, database accesses, file reading or similar).

Here's an example:

```kotlin
fun main() {
  val service = SpeakerService()
  val apiClient = ApiClient(service)

  apiClient.getSpeakers().map { speakers ->
    speakers.map { it.name }
  }.fold(
    ifFailure = { error -> handleFailure(error) },
    ifSuccess = { names -> handleSuccess(names) }
  )
}

fun handleFailure(error: Throwable): Unit = TODO()
fun handleSuccess(speakerNames: List<String>): Unit = TODO()

class ApiClient(private val service: SpeakerService) {
  fun getSpeakers(): Try<List<Speaker>> = Try {
    service.loadAllSpeakers()
  }
}
```

That looks fine at a first glance, since all the exceptions thrown in the HTTP query are automatically captured by `Try`, which you can fold over to apply different effects depending on the case. But this actually has an issue:

> `Try` constructors are eagerly evaluated.

That means you can't compose a declarative computation using it that is not run **as soon as it gets created**. That removes any chances to keep any control over the side effect as is (network query in this case). To make it safe it would require to be already run into a safe environment like a coroutine context so the effect gets suspended.

```kotlin
suspend fun getSpeakers() = apiClient.getSpeakers().map { speakers ->
  speakers.map { it.name }
}.fold(
  ifFailure = { handleFailure(it) },
  ifSuccess = { handleSuccess(it) }
)

class ApiClient(private val service: SpeakerService) {
  suspend fun getSpeakers(): Try<List<Speaker>> = Try {
    service.loadAllSpeakers()
  }
}
```

Here I made the api query suspended, then we call it from a coroutine context or another suspended function. That ensures the effect is suspended and you can't run it in an unsafe scope. That's fine now, and definitely fair to use if it fits well in your codebase.

In my opinion, these "hybrid" styles that combine some FP goodies with other styles like imperative or OOP use to come handy as a reasonable middle step towards migration to pure functional programming.

Another alternative and a more old style solution would be running the code in a separate thread from a pool managed by a `ThreadPoolExecutor`. Both approaches impose a "safety" barrier before the effect point that makes the computation deferred.

### Try deprecation, use Either

If you use `Try` in later versions of Arrow, you'll find out the complete data type is being deprecated ⚠️

![try deprecation image](/assets/images/try_deprecation.png)

The actual deprecation message says:

> "Try will be deleted soon as it promotes eager execution of effects, so it’s better if you work with Either’s suspend constructors or a an effect handler like IO"

So that's pretty much in the line of how people has been using it.

If your intention is to perform side effects under a safe context like the one exposed above, and keep capturing errors, you can use `Either<Throwable, A>` for that. By *"Either's suspend constructors"* we're referring to [its catch contructor functions](https://github.com/arrow-kt/arrow/blob/2fb1b1cc6def8667df51479d1379c01ce1cd6fa8/modules/core/arrow-core-data/src/main/kotlin/arrow/core/Either.kt#L859).

These are the overloads available:

```kotlin
suspend fun <R> catch(f: suspend () -> R): Either<Throwable, R> =
  catch(::identity, f)

suspend fun <L, R> catch(fe: (Throwable) -> L, f: suspend () -> R): Either<L, R> =
  try {
    f().right()
  } catch (t: Throwable) {
    fe(t.nonFatalOrThrow()).left()
  }
```

As you can see both are defined as `suspend fun` and work over `suspended` computations (function parameter) so they ensure two things:
* You're using them in a safe environment.
* They're used to run a side effect.

Moving our api client to `Either#catch` would look like this:

```kotlin
suspend fun getSpeakers() = apiClient.getSpeakers().map { speakers ->
    speakers.map { it.name }
  }.fold(
    ifLeft = { error -> handleFailure(error) },
    ifRight = { names -> handleSuccess(names) }
  )

class ApiClient(private val service: SpeakerService) {
  suspend fun getSpeakers(): Either<Throwable, List<Speaker>> = Either.catch {
    service.loadAllSpeakers()
  }
}
```

Thanks to this we'd keep our computation safe, capturing errors and run in a safe environment, everything imposed by `Either#catch` which is a direct replacement for the `Try` use cases that imply controlling effects.

`Either#catch` lifts errors into its `Left` side and the successful values into its `Right` one.

In case you want to strongly type your errors as soon as they enter your domain you can use the cool second overload that allows you to pass a lambda for mapping the error:

```kotlin
suspend fun getSpeakers(): Either<DomainError, List<Speaker>> = Either.catch(
  fe = { error -> error.toDomain() },
  f = { service.loadAllSpeakers() }
)
```

That will automatically catch any errors and map them into the `Left` side using the mapping function provided, so you flatten your type hierarchy 👏

### Moving into full FP style

We've showcased how to solve the problem of effect control in a "hybrid" scenario where you already got a coroutine context or you're running your effect within a different thread managed by a pool executor or similar.

If you want to go full FP end to end using an *"effect handler"* you'll most likely want to get rid of the hybrid style (coroutines in this case) and use [`IO`](https://arrow-kt.io/docs/effects/io/).

`IO` also automatically captures errors like `Try` or `Either#catch()`, but it defers the computation by default, so you keep control over the effect with no exceptions.

We can remove the `suspend` modifier from everywhere now:

```kotlin
class ApiClient(private val service: SpeakerService) {
  fun getSpeakers(): IO<List<Speaker>> = IO {
    service.loadAllSpeakers()
  }
}
```

And use it in any environment, since `IO<A>` is safe by definition.

```kotlin
apiClient.getSpeakers().map { speakers ->
  speakers.map { it.name }
}.map { names ->
  handleSuccess(names)
}.handleError { error ->
  handleFailure(error)
}
```

This is safe as is because the program is actually not even run yet, **it stays deferred**.

When you stack operations on top of `IO<A>`, the complete stack is deferred. If you are doing pure FP style you will keep stacking operations like that, and then from the edge (entry point) of your program you'll actually run it hence perform all the side effects:

```kotlin
// Our program entry is a main function in this case. The edge of the world.
fun main() {
  val service = SpeakerService()
  val apiClient = ApiClient(service)

  val program = apiClient.getSpeakers().map { speakers ->
    speakers.map { it.name }
  }.map { names ->
    handleSuccess(names)
  }.handleError { error ->
    handleFailure(error)
  }

  // Run the program asynchronously
  program.unsafeRunAsync { callback: Either<Throwable, Unit> ->
    callback.fold(
      ifLeft = { System.out.println("Something failed ${it.message}") },
      ifRight = { System.out.println("Everything went great!") }
    )
  }

// Run the program synchronously
  program.unsafeRunSync()
}
```

So you can run your program both **synchronously and asynchronously** at will. Running it synchronously means it's gonna perform all side effects and **throw in case there's an error**. It's usually better to run it asynchronously so you have the chance to pass a callback to decide how to handle both errors and successful results.

As you've probably observed, both variants are named after *"unsafe"*. That is because **we are at the edge of the world here**, where these functions are thought to be used, so we try to be as explicit as possible about that fact.

At this point we are beyond the safety limits, hence out of the "purity" boundary. Our `program` remains completely pure though, so **our complete architecture is deterministic and easy to reason about, hence much more testable**.

### Strongly typing errors with IO

If you're looking for a way to strongly type errors, I'm afraid the best data type for that is still in the works. It is what is usually called `BIO<E, A>` or `BifunctorIO<E, A>`, and **we will release it before the end of the year**.

`BIO<E, A>` is an equivalent for `IO<A>` but with the power to pick the error type, not have it just fixed to `Throwable` as `IO` does, so it will flatten your effectful hiearchy and allow you to keep strong typing on your errors.

In the mean time my suggestion is to stay with a nested `IO<Either<E, A>>` which is suboptimal but probably much safer than `Try` or `Either#catch` if you're targeting a pure functional style.

But no worries, `BIO<E, A>` is coming pretty soon and I'll write about it so you can have an easy look on how to use it properly 🙏

### Leveling up

If you want to go further and encode your functional program with a direct (and fairly simple) style, but at the same time keep control over your side effects, my best suggestion is to take a look at [ArrowFx](https://arrow-kt.io/docs/effects/fx/). You will be writing pure code with all the pure FP goodies but it will totally feel like if you were writing imperative code.

I will get into that in depth in an upcoming post, so stay tuned!

### Final words

To recap a bit, we'd recommend using `IO` as much as possible when you're working with side effects, since it's safe by definition. It converts an impure computation (side effect) into a pure one by deferring it.

As soon as `BIO` is available, I'll strongly recommend it for effectful computations over any other type.

In case you're targeting a hybrid style were your safety point is earlier in the architecture, like a jump to a coroutine context, then it's safer to use `Try`. But in that case please move your code to use `Either#catch()`, those `Either` suspended constructors are thought for the very same purpose, and `Try` will get removed pretty soon because of redundancy its end goal.

Let me clarify deprecation is something we are advocating for due to the current state of `Try` in the Kotlin developer scene, so please don't take it as a *"`Try` should be removed in all FP libs out there"* statement.

If you’re interested in any topics regarding Functional Programming in Kotlin, feel free to keep an eye on my blog 🙏🏽. You can also [follow me on Twitter](https://twitter.com/JorgeCastilloPR).

Some other links you might like:

* [When Android met FP](https://jorgecastilloprz.github.io/when-android-met-fp)
* [Polymorphic apps using Arrow](https://jorgecastilloprz.github.io/polymorphic-apps-with-arrow)
* [Kotlin Functional Programming: Does it make sense?](https://jorgecastilloprz.github.io/kotlin-fp-does-it-make-sense)
* [Kotlin purity and function memoization](https://jorgecastilloprz.github.io/kotlin-purity-and-function-memoization)
* [Kotlin dependency injection with the Reader Monad](https://jorgecastilloprz.github.io/kotlin-dependency-injection-with-the-reader-monad)
* [Kotlin FP I: Monad Stack](https://jorgecastilloprz.github.io/kotlin-fp-1-monad-stack)
* [Kotlin FP II: Monad Transformers](https://jorgecastilloprz.github.io/kotlin-fp-2-monad-transformers)
