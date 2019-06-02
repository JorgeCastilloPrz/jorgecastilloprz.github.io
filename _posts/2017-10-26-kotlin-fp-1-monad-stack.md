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

### Modeling Error & Success

Almost any system out there requires to access some external sources to fetch data, like databases, APIs, or any type of external caches. I’m not talking about just Android apps.

Those sources of data provide **2 different scenarios in their responses**. The caller can get back a successful bunch of data, or an error, usually presented with an exception. So you have a result which has a clear duality, and you need to find a way to explicitly reflect that in code.

If we think about *Clean Architecture*, you’ll probably remain about those use cases running on recycled threads provided by a `ThreadPool`, where exceptions thrown by external `DataSources` or `Repositories` are catched and then notified to the caller using callbacks.

This approach can be useful for simple apps, but has some inherent problems.

First of all, you are forced to switch from exceptions to callback propagation in terms of errors. That happens because **exceptions are not able to surpass thread limits**.

It also has **referential transparency problems**. Callbacks break it, since you are not able to reflect what the function is going to return just by looking at its return type.

But the main point is that we have 2 different ways to get a result from the method call. And in the end both are parts of a duality fact inside the same operation result, isn’t it? We should be able to reduce this duality to a single possible branch.

You could use `RxJava` to solve the problem by **joining both paths in a single stream**. That could be an interesting way to go here if you are using OOP, since the flow is always reduced to a single stream.

But how could these problems be solved if we want to go for Functional Programming using [Arrow](https://arrow-kt.io)?

We can easily reflect the result duality using the `Either<A, B>` type. `Either` is a disjoint union. **That means it’s always gonna be type `A` or `B`, but never both.**

Behind the scenes, it’s a sealed class with 2 possible implementations: `Left(a: A)` or `Right(b: B)`.

By convention on FP languages, when you present error and success cases using `Either`, **the left side is used for the error type, and the right side for the successful one**.

As you can see, `Either` is the perfect candidate to fulfill our needs. So let’s move to `Kotlin` now and look at how we could benefit from modeling the mentioned duality using `Either<CharacterError, SuperHero>>`:

```kotlin
sealed class CharacterError {
  object AuthenticationError : CharacterError()
  object NotFoundError : CharacterError()
  object UnknownServerError : CharacterError()
}
```

First step: I want to model the domain expected errors using a `sealed` class. After all, you need to **seal a hierarchy of errors supported in your domain** so the rest of your app can react gracefully to anything happening on external sources. Any exceptions being thrown by any external `DataSource` or `Repository` need to be mapped to any of the domain ones.

Lets code now a network `DataSource` implementation to fetch a bunch of super heroes:

```kotlin
/* data source impl */
fun getAllHeroes(service: HeroesService): Either<CharacterError, List<SuperHero>> =
    try {
      Right(service.getCharacters().map { SuperHero(it.id, it.name, it.thumbnailUrl, it.description) })
    } catch (e: MarvelAuthApiException) {
      Left(AuthenticationError)
    } catch (e: MarvelApiException) {
      if (e.httpCode == HttpURLConnection.HTTP_NOT_FOUND) {
        Left(NotFoundError)
      } else {
        Left(UnknownServerError)
      }
    }
```

Here, heroes are fetched using a service and then mapped to domain models. Afterwards, those are being wrapped on a `Right(heroes)` and retuned. It’s the right side for the successful result.

But if anything goes wrong, exceptions are catched and mapped to domain, returning a `Left(error)` in that case.

So the data layer is returning a **completely explicit** result with the type: `Either<CharacterError, List<SuperHero>>`. Just by looking at the method declaration, the caller can simply know that it would be returning either a domain error, or a valid heroes collection. Simple, isn’t it?

```kotlin
// Use case function
fun getHeroes(dataSource: HeroesDataSource, logger: Logger): Either<Error, List<SuperHero>> =
    dataSource.getAllHeroes().fold(
    { logger.log(it); Left(it) },
    { Right(it.filter { it.imageUrl.isEmpty() }) })
```

The use case can also be very straightforward. `Either` has a `fold` operation to fold over its two possible values, so you provide two lambdas for the two different result types.

Depending on the returned value from the `DataSource` (which will be a `Left` or a `Right`), the corresponding lambda will be run.

So we use the error one to log the error and return the successful value as it is, still wrapped on a `Left`. Otherwise, we keep the `Right` wrapping the valid collection but after filtering the non valid heroes out from it. (Like the ones which do not have a valid image url, for example).

So presentation code could look like the following. We can fold again over the already composed computation to apply different effects on the view, depending on the case:

```kotlin
fun getSuperHeroes(view: SuperHeroesListView, logger: Logger, dataSource: HeroesDataSource) {
  getHeroesUseCase(dataSource, logger).fold(
      { error -> drawError(error, view) },
      { heroes -> drawHeroes(heroes, view) })
}

private fun drawError(error: CharacterError,
    view: HeroesView) {
  when (error) {
    is NotFoundError -> view.showNotFoundError()
    is UnknownServerError -> view.showGenericError()
    is AuthenticationError -> view.showAuthenticationError()
  }
}

private fun drawHeroes(success: List<SuperHero>, view: SuperHeroesListView) {
  view.drawHeroes(success.map {
    RenderableHero(
        it.name,
        it.thumbnailUrl)
  })
}
```

As you see, we are passing dependencies manually all the way down as function parameters. That has an explanation.

On Functional Programming, you don’t use instances most of the time for intermediate operations about transforming / manipulating data. You have those operations + value types like the errors or successful heroes list.

You might have not noticed yet, but **all the functions I have been showing on this `Either` example, are defined at a package level**. They do not belong to any instance. That’s because on FP, you try to play with pure functions with no side effects, so those functions do not have any need to live under an enclosing class, since there is no shared state and they have not allowed to access external state. Pure functions get a bunch of parameters (dependencies) and provide a result using those. That’s all.

So, on FP, dependencies are passed as function parameters. And yes, a little bit ahead on this article we will find a way to get rid of them. (DI. Yay! 🎊)

If you want to know more about error handling strategies using [Arrow](https://arrow-kt.io/), please take a look at [this section on the official documentation](https://arrow-kt.io/docs/patterns/error_handling/).

---

![tibet monastery 2](assets/images/tibet_monastery2.jpeg)

So, our magnificent temple is starting to appear in front of our eyes!. We already have some foundations ready. Maybe some walls to? Let’s keep moving, since winter is coming. ❄️

### Async + Threading

You probably noticed that we are ignoring asynchrony and threading for the time being. But we will need to find an approach to cover that.

Every time we render something on screen or make a query to an external source of data, what we are really doing is an **IO computation**. That computation is a side effect, so that does not play a good role inside our FP approach. We want to go pure starting on the presentation layer and all the layers beyond that one, so we need to do something at least for the `DataSource` calls.

That’s where the **IO Monad** comes into play. Please, forget about the “M” word right now. Being completely honest, that is not needed to understand the approach and will be easier for you if we keep that aside for now. I obviously know what it is and you also will at the end of this series.

**IO wraps a side effect**, a computation, and makes it pure. That’s because it is still not run, just deferred to the moment when we finally decide to execute it and perform its unsafe effects.

`IO` is well known and very important in Haskell, for example. Since side effects are not even allowed in the language!. Thanks to `IO`, we can keep growing our **Monad Stack** here, also making that computation explicit in the return types of the call tree.

So let’s upgrade our network `DataSource` implementation:

```kotlin
/* network data source implementation */
fun getAllHeroes(service: HeroesService, logger: Logger):
IO<Either<CharacterError, List<SuperHero>>> =
    runInAsyncContext(
        f = { queryForHeroes(service) },
        onError = { logger.log(it); it.toCharacterError().left() },
        onSuccess = { mapHeroes(it).right() },
        AC = IO.asyncContext()
    )
```

The `runInAsyncContext()` is just a function we created for syntax. We are using it to run a lambda inside a **coroutine** and lift that computation into the context of `IO`, which means we will be wrapping it into `IO<A>`.

We provide a bunch of parameters to it:

* `f`: The function to be run inside the coroutine.
* `onError`: The lambda to run in case the computation throws an exception when it gets executed. We can log errors here and then map the throwable into a domain `CharacterError` using an extension function. Then we raise the already mapped error into a `Left(e)` and return it.
* `onSuccess`: The lambda executed when the query succeeds. We map the heroes DTO collection to a collection of domain models and then raise it to a `Right(heroes)` before returning it.
* `AC`: An `AsyncContext`, which is a typeclass to move data from an uncontrolled callback constrained context to a type supporting asynchrony like `IO`.

As you can see, the return type of the function is completely explicit about the computation being done:

* `IO<Either<CharacterError, List<SuperHero>>>`

That means the `DataSource` will be returning an `IO` computation that when it gets run it will return Either a `CharacterError`, or a valid `List<SuperHero>`. Semantically **the type speaks by itself**.

So lets look at the use case, which is a bit special now:

```kotlin
/* Use case */
fun getHeroesUseCase(service: HeroesService, logger: Logger):
IO<Either<CharacterError, List<SuperHero>>> =
    getAllHeroesDataSource(service, logger).map { it.map { discardNonValidHeroes(it) } }
```

The use case function needs to call map twice. That’s because now we have 2 nested monads on the result value from the heroes `DataSource: IO<Either...>>`. So we map over the `IO` instance, to be able to map over the `Either` afterwards. That does not mean the computations are being unwrapped and executed, we are just declaratively composing the stack of operations using monads. **Everything keeps deferred**.

We will simplify this double mapping on the following posts with an interesting style which presents the following natural iteration after the **Monad Stack**. You will see how can we overcome this in a very natural way. by collapsing the whole stack in a single type.

It’s nice to notice that **Either<A, B> is right biased**. That means functions like `map` or `flatMap` always apply over it’s right side. The left one is always kept as it is.

Finally, we arrive to our presentation layer. We will ask IO to perform the unsafe effects on this layer, even though we know that’s not ideal, since those are side effects. But just for iteration purposes:

```kotlin
/* Presentation logic */
fun getSuperHeroes(view: SuperHeroesListView, service: HeroesService, logger: Logger) =
    getHeroesUseCase(service, logger).unsafeRunAsync { it.map { maybeHeroes ->
      maybeHeroes.fold(
          { error -> drawError(error, view) },
          { success -> drawHeroes(success, view) })}
    }
```

Here we have the same thing we previously had, but this time we tell `IO` to perform its effects. So the `unsafeRunAsync()` call is ordering `IO` to finally unwrap and execute it’s computation, and apply the resulting value to the lambda afterward.

So the view is now capable of rendering errors or heroes depending on that result.

But can iterate on this further. Ideally we would **push the effects to a single place at the edge of the system**, which on Android is the overriden methods in the Activity or View and in non Android applications would be the endpoints or main methods.

That is the place where purity becomes unsafe effects, since Android in the end plays with shared state and things need to get rendered on screen. We try to push those potential problems to the outer most layer and keep the whole architecture design based on purity.

To achieve this, we just need to apply what it’s called **lazy evaluation**. We just need to defer all the functions on the call tree. We can do that by returning functions instead of already computed values, as we explained on the previous post.

So we can compose our complete execution tree to be like (pseudocode):

* *presenter(deps) = { deps -> useCase(deps) }*
* *useCase(deps) = { deps -> dataSource(deps) }*
* *dataSource(deps) = { deps -> deps.apiClient.fetchHeroes() }*

So each level can return a function instead of the already computed value. The function will be able to perform a computation just when some dependencies are passed to it, but **not before**. In the end, the `DataSource` can pick the required dependencies to do its work.

So when the view implementation calls the Presenter / ViewModel, instead of returning an already computed value (which would end up calling the API and performing effects), it would be returning a function capable of computing the result. So then it’s the view implementation the layer deciding about when to pass the required dependencies in to finally unfold the whole execution tree.

But passing dependencies manually all the way down can be painful. Don’t we have a way to automatically do that?

Obviously we are talking about **Dependency Injection** now.

---

![tibet monastery 3](assets/images/tibet_monastery3.jpeg)

We already have all the walls up now. There is still no roof, but we are almost there. At least, wild monks will not be able to sneak into the temple silently during the night anymore 👏👏, and we could start thinking about living inside, even if we could get a bad cold 😓😓.

May be we just need to iterate a bit more!

### Dependency Injection

For Dependency Injection we are gonna be using something with a quite weird name: The **Reader Monad**. [I already wrote some lines about it’s most basic usage and concept](https://jorgecastilloprz.github.io/kotlin-dependency-injection-with-the-reader-monad) which you might want to read before this new iteration.

I want to use the `Reader` because I’m composing a monad stack here, so it fits perfectly well.

**The Reader wraps a computation with the type `(D) -> A`, and enables composition over computations with that type.**

`D` stands for the *“Reader context”*, and it represent the dependencies needed for the computation to run. So those computations are exactly like the ones we have on each layer of our architecture at the moment, aren’t they?

It also automatizes dependency passing all the way down, since it does it by itself thanks to the way `Readers` compose all together.

So it’s fixing the two concerns we still needed to solve!. It:

* Defers computations at all levels, since it wraps computations (functions waiting for dependencies to be passed in).
* “Injects dependencies” by automatically passing those across the different function calls, so we don’t need to do it manually by ourselves.

So the idea is to wrap the return types we had until now with `Reader`. So here you have the DataSource implementation one more time:

```kotlin
/* data source could look like this */
fun getHeroes():
Reader<GetHeroesContext, IO<Either<CharacterError, List<SuperHero>>>> =
    Reader.ask<GetHeroesContext>().map({ ctx ->
      runInAsyncContext(
          f = { ctx.apiClient.getHeroes() },
          onError = { it.toCharacterError().left() },
          onSuccess = { it.right() },
          AC = ctx.threading
      )
    })
```

Don’t be scared. I know, we are starting to get a bit lost on types now. But as I said before, we will fix that on the next post on the series. Trust me for now on! 🤗

You probably noticed that our computation is still there as it was before, but now it’s done after mapping over a `Reader`. But where is this `Reader` coming from? Seems like it’s being statically summoned or something, doesn’t it?

If you look at the beginning of the function body, you will find the statement:

```kotlin
Reader.ask<GetHeroesContext>().map { ctx -> ... }
```

`GetHeroesContext` is `D` here, the `Reader` context. That’s just a data class I use to provide all the required dependencies. The context will be instantiated and passed in the moment you want to run the whole computation tree, not before.

Those dependencies on `D` are be the ones I need to inject for the complete call chain. In other words, `D` is equivalent to one of those `Dagger` dependency graphs / components with all the bindings that we usually build per activity or application.

The `ask()` call is part of the `Reader` companion object, so we can call it statically, and returns an already lifted `ReaderT` from nowhere, wrapping a computation with the type `{(D) -> D}`. So we can map over that `Reader` to get access to the context `D`, which contains all the dependencies. That’s why we can use those inside of the lambda.

So the return type is now:

* `Reader<GetHeroesContext, IO<Either<Error, List<SuperHero>>>>`

This type still needs a further iteration to collapse the big nested type on a single one. That’s what any FP developer would do with a big monad stack. But it’s still soon to showcase that. 😉

Please, take a look at what the type is saying here:

It is deferring a computation (`Reader`) which will be waiting to get some dependencies passed in to run (`GetHeroesContext`). At that moment it will be able to perform an IO computation that could return `Either` an `Error` or a valid `List<SuperHero>`.

If we move a step back on the call chain, we can see how the use case keeps doing the same it was doing before, no changes needed. I have removed the return type here just to reduce noise, but please append it if possible. It’s good to be explicit in terms of return types.

```kotlin
/* use case */
fun getHeroesUseCase() = fetchAllHeroes().map { io ->
  io.map { maybeHeroes ->
    maybeHeroes.map { discardNonValidHeroes(it) }
  }
}
```

Also, the presenter code is really similar, but now we also lift up a reader with the context inside before doing anything else. That’s the way for us to automatically get access to the context containing the dependencies:

```kotlin
/* presenter code */
fun getSuperHeroes() = Reader.ask<GetHeroesContext>().flatMap(
{ (_, view: SuperHeroesListView) ->
  getHeroesUseCase().map({ io ->
    io.unsafeRunAsync { it.map { maybeHeroes ->
        maybeHeroes.fold(
            { error -> drawError(error, view) },
            { success -> drawHeroes(view, success) })
      }
    }
  })
})
```

This time, we do something interesting, we apply **destructuring** over the context, since it’s a data class. The context is still there:

`(_, view: SuperHeroesListView) -> ...`

Since we just need access to one of the dependencies, which would be the MVP view contract, we can apply deconstruction to get quick access to it.

The rest of the function keeps doing the same it was doing before. But this time **we are returning a `Reader` wrapping the computation**. View implementation can now call the presentation pure function, get a computation back from it, and keep the choice to run it when dependencies are ready at a later moment.

```kotlin
/* we perform unsafe effects on view impl now */
override fun onResume() {
  /* presenter call */
  getSuperHeroes().run(heroesContext)
}
```

Here, it’s time to pass in the `Reader` context instance we want to use. 👏

Thanks to this, we were able to push effects out from presenter to the view implementation and now we have the whole execution tree completely pure. No side effects, no state. That’s a big win.

**If you needed to swap dependencies at testing time at any point on the dependency tree, you wouldn’t need complex frameworks or libraries to do so. You would just need to provide a different context instance that could extend from the real one and mock any required instances inside of it by extension, the same way Mockito does.**

---

### Some Conclusions

A properly built **Monad stack** can solve all the concerns any Android app could have over the years. Those concerns and the approach used to resolve them are completely generic problems like error handling, Dependency Injection or IO computations, and so on.

Those strategies can be shared with any other apps you write in the future. Those can also be shared with any other platforms that are able to run Kotlin. And if they can’t, they can always use FP and use the same exact approaches.

I realize that the Monad Stack could be a bit hard to understand in the beginning for some people, since we are not very used to this new paradigm as Android developers, the same way we weren’t very used to RxJava in the beginning.

**That does not necessarily mean it’s a bad thing or it can’t succeed**. It provides very interesting benefits like the ones we explained on the previous post, so it deservers a try and some investigation at least. I suggest you to do so and get into this, since you will notice you are a better developer when you end up understanding it, as I did. 😊

So big hugs for readying this **TL;DR** and stay tuned for the next chapter, where we will use something really cool called **Monad Transformers** to simplify the nested types! That’s what a real Functional Programing dev would do to iterate on this, most likely.

Remember that you can [add me on Twitter](https://www.twitter.com/JorgeCastilloPr) to know more about this topic and many more.

---

*“So you ended up building your magnificent temple. Feel the air, touch the sky. Now step in and live on it peacefully.”*

![tibet monastery 4](assets/images/tibet_monastery4.jpeg)
