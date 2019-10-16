---
layout: post
current: post
cover: assets/images/painting2.jpg
navigation: True
title: Please Try to use IO!
date: 2019-10-16 12:36:00
tags: [kotlin, fp]
class: post-template
subclass: 'post'
author: jorge
---

Let's dive into why we consider `IO` safer than `Try` and why we recommend using it to control your effects.

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

That means you can't compose a declarative computation using it that is not run **as soon as it gets created**. That removes any chances to keep any control over the side effect as is (network query in this case). To make it safe it would require to be already run into a safe environment like a coroutine context.

```kotlin
suspend fun getSpeakers() = repository.getAllSpeakers().map { speakers ->
  speakers.map { it.name }
}.fold(
  ifFailure = { error -> handleFailure(error) },
  ifSuccess = { names -> handleSuccess(names) }
)

class SpeakersRepository(private val apiClient: ApiClient) {
  suspend fun getAllSpeakers(): Try<List<Speaker>> {
    return apiClient.getSpeakers()
  }
}

class ApiClient(private val service: SpeakerService) {
  fun getSpeakers(): Try<List<Speaker>> = Try {
    service.loadAllSpeakers()
  }
}
```

Here I've introduced a `SpeakersRepository` that makes the computation suspended. Then we call it from a coroutine context which is prepared to run suspended effects. That's fine now, and definitely fair to use if it fits well in your codebase.

In my opinion, these "hybrid" styles that combine some FP goodies with other styles like imperative or OOP use to come handy as a reasonable middle step towards migration to pure functional programming.

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

Thanks to this we'd keep our computation safe, capturing errors and run in a safe environment, everything imposed by `Either#catch`.

In case you want to strongly type your errors as soon as they enter your domain you can use the second variant:

```kotlin
suspend fun getSpeakers(): Either<DomainError, List<Speaker>> = Either.catch(
  fe = { error -> error.toDomain() },
  f = { service.loadAllSpeakers() }
)
```

That will automatically catch any errors and map them into the `Left` side using the mapping function provided, so you flatten your type hierarchy.

### Moving into full FP style

If you want to go full FP end to end you'll most likely want to get rid of the hybrid style (coroutines in this case) and use [`IO`](https://arrow-kt.io/docs/effects/io/).

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

That is because the program is actually not even run yet. When you stack operations on top of `IO<A>`, the complete stack is deferred. If you are doing pure FP style you will keep stacking operations like that, and then from the edge (entry point) of your program you'll actually run it hence perform all the side effects:

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
}
```

### Final words

To sumarize and recap a bit, we'd recommend using `IO` as much as possible when you're working with side effects, since it's safe by definition. It converts an impure computation (side effect) into a pure one by deferring it.

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
