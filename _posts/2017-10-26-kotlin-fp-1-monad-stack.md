---
layout: post
current: post
cover: assets/images/tibet_monastery.jpeg
navigation: True
title: Kotlin Functional Programming I - Monad Stack
date: 2017-10-26 10:18:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

Functional Programming is about encoding concerns into types. Learn how to create an end to end architecture using them.

*“You could touch the sky by building up your own temple, floor by floor, solving problems with mindfulness. Then you could live on that temple forever.”*

Peaceful, isn’t it? And also true.

If you haven’t read [the first post of this series](https://jorgecastilloprz.github.io/kotlin-fp-does-it-make-sense) yet, I would suggest you to do it, since some concepts explained there will be used on this post.

---

When we think about Functional Programming, we must understand one key concept: **Problems can be solved once, and then reuse that solution forever**.

**When you code FP, problems solved are usually generic ones. Those are not bounded to any semantics or implementation details**. Just to give you some examples, we could be talking about global concepts like asynchrony, IO, threading, dependency injection, or the way to replace dependencies by mocks on different nested levels of your architecture.

So those problems can be key pieces on any system. Ideally you should just need to work on implementing a solution once. An already resolved DI approach can be used on any app you write, or even on different platforms like backend, in case they also use Kotlin.

So, to reflect this in a very transparent way, we will code our own app architecture, moving on step by step, to end up composing its complete stack.

> Disclaimer: You can encode more efficient architectures that do not require stacking monads, but I believe the reasoning we'll go through on this post will be valuable as a previous step towards the mentioned approaches. Here, we will learn how to encode all our program's concerns into the data types.

### Modeling Error & Success

Almost any system out there requires to access some external sources to fetch data, like databases, APIs, or any type of external caches. I’m not talking about just Android apps.

Those sources of data always provide at least **2 different scenarios in their responses**. The caller can get back a successful data, or an error, usually presented with an exception. So you have a result which has a potential duality, and you need to find a way to explicitly reflect that in code.

Some years ago *Clean Architecture* became very popular in the Android world. There was a very common approach to encode error handling in threads using exceptions (there wasn't coroutines back then). We usually threw exceptions from outer layers like the data one, then inner layers required to capture those.

This approach was probably enough for simple apps back then, but it had some (big) inherent issues.

First of all, you were forced to switch from exceptions to callback propagation in terms of errors. That happened because **exceptions are not able to surpass thread limits**. The thread just swallows them.

It also has important **referential transparency problems**. Callbacks break it, since you are not able to reflect what the function is going to return just by looking at its return type. You need to click in the callback and look inside to see what's it able to return.

But the main point is that we have **2 different paths to get a result from the method call**. But at the end of the day both paths represent a duality occurring for the same operation result, isn’t it? We should be able to reduce this duality to a single possible path to reduce code branching.

You could use `RxJava` to solve the problem by **joining both paths in a single stream**. That could be an interesting way to go here if you wanted to use reactive programming, which could be considered an intermediate step towards a more functional style.

But how could these problems be solved if we want to go for pure Functional Programming using [Arrow](https://arrow-kt.io)?

We can easily reflect the result duality using the `Either<A, B>` type. `Either` is a disjoint union. **That means it’s always gonna be `A` or `B`, but never both.**

Behind the scenes, it’s a sealed class with 2 possible implementations: `Left(a: A)` and `Right(b: B)`.

By convention on FP languages, when you present error and success cases using `Either`, **the left side is used for the error type, and the right side for the successful one**.

As you can see, `Either` is the perfect candidate to fulfill our needs. So let’s look at how we could benefit from modeling the mentioned duality using `Either`.

Let's say we've got a data layer operation that can return either an error or a valid `SuperHero` succesfully fetched from a network service.

We could make the result type be `Either<CharacterError, SuperHero>>`, given the following implementation for `CharacterError`:

```kotlin
sealed class CharacterError {
  object AuthenticationError : CharacterError()
  object NotFoundError : CharacterError()
  object UnknownServerError : CharacterError()
}
```

First step: I want to model the domain expected errors using a `sealed` class. After all, you need to **seal a hierarchy of errors supported in your domain** so the rest of your app can react gracefully to anything happening on external sources. Any exceptions being thrown by any external `DataSource` need to be mapped to one of the domain ones.

Lets code now a network `DataSource` implementation to fetch a bunch of super heroes using the mentioned pattern:

```kotlin
/* data source impl */
fun getAllHeroesDataSource(service: HeroesService, gson: Gson): Either<CharacterError, List<SuperHero>> =
    try {
        val response = service.getCharacters()
        if (response.isSuccessful) {
            val networkHeroes = gson.deserializeHeroes(response.body)
            Right(networkHeroes.toDomain())
        } else {
            when (response.code) {
                401 -> Left(CharacterError.AuthenticationError)
                404 -> Left(CharacterError.NotFoundError)
                else -> Left(CharacterError.UnknownServerError)
            }
        }
    } catch (e: NetworkErrors.Unauthorized) {
        Left(CharacterError.UnknownServerError)
    }
```

Here, heroes are fetched using a service and deserialized to network models. If everything works alright, we map them to domain models before returning them to the caller (probably our domain layer). In this scenario we wrap the result into `Right(heroes)` given the operation succeeded and we are returning valid data.

In the other hand, in case there's any error we always map the error to the corresponding domain error, then wrap it into `Left(error)`. As we previously said, by an FP convention we use the left side of `Either` for the errors.

So the data layer is **very explicit** about what it could return: `Either<CharacterError, List<SuperHero>>`. Just by looking at the method declaration, the caller can simply know that it would be returning either a domain error, or a valid heroes collection. Simple, isn’t it?

Let's move on to our domain layer now. Here's how it could look:

```kotlin
fun getHeroesUseCase(service: HeroesService, gson: Gson, logger: Logger): Either<CharacterError, List<SuperHero>> =
    getAllHeroesDataSource(service, gson).fold(
        { error -> logger.log(error); Left(error) },
        { heroes -> Right(heroes.filter { it.thumbnail.isEmpty() }) })
```

The use case can also be very straightforward. `Either` has a `fold` operation to fold over its two possible values, so **you can provide two lambdas to cover the two potential result types**. One thing to notice is `fold()` enforces to return something, so both lambdas need to map to the same result time. That makes our codebase more functional in a way, since we'll not apply any computations without returning a result.

Depending on the returned value from the `DataSource` (which will be a `Left` or a `Right`), the corresponding lambda will be run.

So we use the error one to log the error and return the successful value as it is, still wrapped on a `Left`. Otherwise, we keep the `Right` wrapping the valid collection but after filtering the non valid heroes out from it, following our domain rules. Let's say that's our only business logic here, for the sake of the example.

Let's see how the presentation logics could look now. We can fold again over the already composed computation to apply different effects on the view, depending on the case:

```kotlin
fun getSuperHeroesPresentation(view: HeroesView, service: HeroesService, gson: Gson, logger: Logger) {
    getHeroesUseCase(service, gson, logger).fold(
        { error -> drawError(error, view) },
        { heroes -> drawHeroes(heroes, view) })
}

private fun drawError(
    error: CharacterError,
    view: HeroesView
) {
    when (error) {
        is NotFoundError -> view.showNotFoundError()
        is UnknownServerError -> view.showGenericError()
        is AuthenticationError -> view.showAuthenticationError()
    }
}

private fun drawHeroes(success: List<SuperHero>, view: HeroesView) {
    view.drawHeroes(success.map {
        SuperHeroViewState(it.name)
    })
}
```

And we got our architecture complete.

But it's not super neat, is it?. As you can see, we are passing dependencies manually all the way down as function parameters. That has an explanation.

You might have not noticed yet, but **all the functions I have been showing on this `Either` example, are defined at a package level**. They do not belong to any instance. That’s because on FP, you try to play with pure functions with no side effects, so those functions do not have any need to live under an enclosing class, given there is no shared state and they are not allowed to access any sort of external state. Pure functions get their parameters (dependencies) passed in, and provide a result using those.

So, on FP, **dependencies are passed as function parameters**. If you find it convoluted or suboptimal don't worry, a little bit ahead we will find a way to get rid of them. (Yay! 🎊)

If you want to know more about error handling strategies using [Arrow](https://arrow-kt.io/), please take a look at [this section on the official documentation](https://arrow-kt.io/docs/patterns/error_handling/).

---

![tibet monastery 2](assets/images/tibet_monastery2.jpeg)

So, our magnificent temple foundations are starting to get built. Maybe some walls to? Let’s keep moving, winter is coming! ❄️

### Async

You probably noticed that we are ignoring asynchrony for the time being, but if we wanted to run this code in Android we'd most likely want to avoid blocking the main thread and making it async. We'll need to find an approach for that.

Every time we render something on screen or make a query to an external source of data, what we are really doing is an **IO computation**. That computation is a side effect, so that does not play a good role inside our FP approach. We want to go pure starting on the presentation layer and all the layers beyond that one, so we need to do something at least for the `DataSource` calls.

That’s where the **IO Monad** comes into play. Don't fear much the word Monad here. Trust me, you'll understand. I don't think that's needed to understand the approach and will be easier for you if we keep that aside for now.

**IO is a data type that wraps a side effect**, (an effectful computation), and makes it pure. That’s because the wrapped effect remains deferred, **still not run**. It's awaiting for the moment when we finally decide to execute it and perform its unsafe effects.

Thanks to `IO` we can make the need for asynchrony and purity explicit in the return types of our architecture.

So let’s upgrade our network `DataSource` implementation so it encodes that concern:

```kotlin
fun getAllHeroesDataSource(service: HeroesService, gson: Gson): IO<Either<CharacterError, List<SuperHero>>> =
    IO {
        val response = service.getCharacters()
        if (response.isSuccessful) {
            val networkHeroes = gson.deserializeHeroes(response.body)
            Right(networkHeroes.toDomain())
        } else {
            when (response.code) {
                401 -> Left(CharacterError.AuthenticationError)
                404 -> Left(CharacterError.NotFoundError)
                else -> Left(CharacterError.UnknownServerError)
            }
        }
    }.handleError { Left(CharacterError.AuthenticationError) }
```

As you can see, the return type of the function is completely explicit about the computation being done:

* `IO<Either<CharacterError, List<SuperHero>>>`

That means the `DataSource` will be returning an `IO` computation that **when it gets run** it will return Either a `CharacterError`, or a valid `List<SuperHero>`. Since it's `IO`, the side effect inside becomes pure by getting deferred. Semantically **the type speaks by itself**.

Also note that **`IO` is able to capture throwable errors automatically for you**, so we get those handled with our error handling strategy attached to the computation through `handleError {}` combinator. The other HTTP errors are not exceptions but automatically captured by `OkHttp`, so we need to control and map those by ourselves.

So lets look at the use case, which is a bit special now:

```kotlin
/* Use case */
fun getHeroesUseCase(service: HeroesService, gson: Gson): IO<Either<CharacterError, List<SuperHero>>> =
    getAllHeroesDataSource(service, gson).map { maybeHeroes ->
        maybeHeroes.flatMap { heroes ->
            Right(heroes.filter { it.thumbnail.isEmpty() })
        }
    }
```

The use case function needs to call map first, then flatMap. That’s because now we have 2 nested monads on the result value from the heroes `DataSource: IO<Either...>>`. So we map over the `IO` instance, to be able to flatMap over the inner `Either` afterwards. That does not mean the computations are being unwrapped and executed, we are just declaratively composing the stack of operations using monads. **Everything keeps deferred**.

It’s nice to notice that **Either<A, B> is right biased**. That means functions like `map` or `flatMap` always apply over it’s right side. The left one is always kept as it is. And the same happens for `IO`, which is biased towards its successful implementation. Keeping that in mind, the code in the previous snippet will just run whenever the computations were successful.

Finally, we arrive to our presentation layer. We will ask IO to perform the unsafe effects on this layer, even though we know that’s not ideal, since those are side effects. But just for iteration purposes:

```kotlin
/* Presentation logic */
fun getSuperHeroes(view: HeroesView, service: HeroesService, gson: Gson) =
    getHeroesUseCase(service, gson).unsafeRunAsync { it.map { maybeHeroes ->
        maybeHeroes.fold(
            { error -> drawError(error, view) },
            { success -> drawHeroes(success, view) })}
    }
```

Here we have the same thing we previously had, but this time we tell `IO` to perform its effects. So the `unsafeRunAsync()` call is ordering `IO` to finally unwrap and execute it’s computation **in an asynchronous way**. Then we can apply our side effects for both possible scenarios: error or success.

So the view is now capable of rendering errors or heroes depending on that result.

But can iterate on this further. Ideally we would **push the effects to a single place at the edge of the system**, or what we use to call "the edge of the world". On Android that would be the `Application`, `Activity`, `Fragment` or even a `CustomView`. In non Android applications it could be the Controller endpoints (for a Resful API) or `main()` methods (for JVM programs).

That is the place where purity becomes unsafe effects, since Android in the end plays with shared state and things need to get rendered on screen. We are already outside of the pure boundary which starts in our presenter. We try to push those potential problems to the outer most layer so we can **keep the whole architecture design end to end based on purity**.

To achieve this we just need to also return `IO` from our presentation logic, so we can push the application of those side effects to the view implementation. So we could refactor the presentation logics to be like:

```kotlin
fun getSuperHeroes(view: HeroesView, service: HeroesService, gson: Gson): IO<Unit> =
    getHeroesUseCase(service, gson).map { maybeHeroes ->
        maybeHeroes.fold(
            { error -> drawError(error, view) },
            { success -> drawHeroes(success, view) })
    }
```

So we're composing those logics on top of the already existing ones, but **they remain pure** since it's still an `IO` computation. The presenter logic is now a deferred side effect (`IO<Unit>`) so as soon as it runs it'll perform the required computations and follow with the corresponding effects over the view.

The view implementation (the edge of the world) will decide when to run it.

So this is cool, but passing dependencies manually all the way down can be painful indeed. Don’t we have a way to automatically do that?

---

![tibet monastery 3](assets/images/tibet_monastery3.jpeg)

We already have all the walls up now. There is still no roof, but we are almost there. At least, wild monks will not be able to sneak into the temple silently during the night anymore 👏👏, and we could start thinking about living inside, even if we could get a bad cold 😓😓.

May be we just need to iterate a bit more!

### Dependency Injection

For Dependency Injection we are gonna be using something the **Reader Monad**. [I already wrote some lines about it’s most basic usage and concept](https://jorgecastillo.dev/kotlin-dependency-injection-with-the-reader-monad) which you might want to read before this new iteration.

**The Reader wraps a computation with the type `(D) -> A`, and enables composition over computations with that type.**

`D` stands for the *“Reader context”*, and it represent the dependencies needed for the computation to run. It also takes care of implicitly passing dependencies all the way down the execution chain.

So the idea is to wrap the return types we had until now with `Reader`. As you can see the type stack keeps growing and also the complexity around it. We'll need to find ways to shortcut that in the future.

Here's how the `DataSource` would look like using the `Reader`:

```kotlin
fun getAllHeroesDataSource(): Reader<DependencyGraph, IO<Either<CharacterError, List<SuperHero>>>> =
    ReaderApi.ask<DependencyGraph>().map { ctx ->
        IO {
            val response = ctx.service.getCharacters()
            if (response.isSuccessful) {
                val networkHeroes = ctx.gson.deserializeHeroes(response.body)
                Right(networkHeroes.toDomain())
            } else {
                when (response.code) {
                    401 -> Left(CharacterError.AuthenticationError)
                    404 -> Left(CharacterError.NotFoundError)
                    else -> Left(CharacterError.UnknownServerError)
                }
            }
        }.handleError { Left(CharacterError.AuthenticationError) }
    }
```

The computation didn't vary, but now it’s done after mapping over a `Reader`. Note how we don't need to pass dependencies as function arguments anymore. The `Reader` will do that implicitly for us.

Here's how we're getting that initial reader to get access to the dependency context:

```kotlin
ReaderApi.ask<DependencyGraph>().map { ctx -> ... }
```

`DependencyGraph ` is `D` here, the `Reader` context. That’s just a data class I use to provide all the required dependencies. The context will be instantiated and passed in the moment you want to run the whole computation tree, not before.

Those dependencies on `D` are be the ones I need to inject for the complete architecture. In other words, `D` is equivalent to one of those `Dagger` dependency graphs / components with all the bindings that we usually build per activity or application.

The `ask()` call is a handy way to get a `Reader` out of a context, so we can get direct declarative access to the context by `mapping` or `flatMapping` it. That's how we get access to the context to get the dependencies from it.

So the return type is now:

* `Reader<GetHeroesContext, IO<Either<Error, List<SuperHero>>>>`

The type stack keeps growing. In further posts we'll learn how we can shortcut all these behaviors and concerns and collapse the return types to make it much simpler.

This return type:

Is a computation that will run whenever some dependencies are ready, never before (`Reader`). As soon as it runs it will be able to perform an IO computation that could return `Either` an `Error` or a valid `List<SuperHero>`. Our type stack is very explicit about the computation nature at every possible level.

If we move a step back on the call chain, we can see how the use case keeps doing the same it was doing before. This time we need to map over the `Reader` returned by the `DataSource`, so we can get access to the inner `IO` computation. Note how we don't need to pass dependencies as function arguments once again.

```kotlin
/* use case */
fun getHeroesUseCase(): Reader<DependencyGraph, IO<Either<CharacterError, List<SuperHero>>>> =
    getAllHeroesDataSource().map { io ->
        io.map { maybeHeroes ->
            maybeHeroes.flatMap { heroes ->
                Right(heroes.filter { it.thumbnail.isEmpty() })
            }
        }
    }
```

Also, the presenter code is really similar, but now we also lift up a reader with the context inside before doing anything else. That’s the way for us to automatically get access to the context containing the dependencies:

```kotlin
/* presenter code */
fun getSuperHeroes(): Reader<DependencyGraph, IO<Unit>> =
    ReaderApi.ask<DependencyGraph>().flatMap { ctx ->
        getHeroesUseCase().map { io ->
            io.map { maybeHeroes ->
                maybeHeroes.fold(
                    { error -> drawError(error, ctx.view) },
                    { success -> drawHeroes(success, ctx.view) })
            }
        }
    }
```

Same thing one more time. We return a `Reader` that wraps a computation that assumes the required dependencies will be available to run it. The computation is able to load the heroes and apply the required effects over the view. The view implementation will take the lead now on actually unfolding the whole execution tree and to apply all the unsafe effects:

```kotlin
/* we perform unsafe effects on view impl now */
fun onResume() {
    super.onResume()
    val dependencies = DependencyGraph(this, HeroesService(), Gson())
    val algebra = getSuperHeroes().run(dependencies).fix()
    algebra.extract().unsafeRunAsync {}
}
```

Here, it’s time to pass in the `Reader` context instance we want to use to provide our dependencies. Then we can finally run the whole architecture asynchronously. 👏

Thanks to this, we were able to push effects out from presenter to the view implementation and now we have the whole execution tree completely pure. No side effects, no state. That’s a big win.

**If you needed to swap dependencies at testing time at any point on the dependency tree, you wouldn’t need complex frameworks or libraries to do so. You would just need to provide a different context instance that could extend from the real one and mock any required instances inside of it by extension, the same way Mockito does.**

---

### Some Conclusions

A properly built **Monad stack** can solve all the concerns any Android app could have over the years. Those concerns and the approach used to resolve them are completely generic problems like error handling, Dependency Injection or IO computations, and so on.

Those strategies can be shared with any other apps you write in the future. Those can also be shared with any other platforms that are able to run Kotlin. And if they can’t, they can always use FP and use the same exact approaches.

I realize that the Monad Stack could be a bit hard to understand in the beginning for some people, since we are not very used to this new paradigm as Android developers, the same way we weren’t very used to RxJava in the beginning.

**That does not necessarily mean it’s a bad thing or it can’t succeed**. It provides very interesting benefits like the ones we explained on the previous post, so it deservers a try and some investigation at least. I suggest you to do so and get into this, since you will notice you are a better developer when you end up understanding it, as I did. 😊

So thanks for reading this **TL;DR** and stay tuned for the next chapter, where we will use something really cool called **Monad Transformers** to simplify the nested types hell we got into!

Remember that you can [add me on Twitter](https://www.twitter.com/JorgeCastilloPr) to know more about this topic and many more.

---

*“So you ended up building your magnificent temple. Feel the air, touch the sky. Now step in and live on it peacefully.”*

![tibet monastery 4](assets/images/tibet_monastery4.jpeg)
