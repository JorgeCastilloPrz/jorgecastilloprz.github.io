---
layout: post
current: post
cover: assets/images/transformer.jpeg
navigation: True
title: Kotlin Functional Programming II - Monad Transformers
date: 2017-11-11 18:18:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

Step inside to understand how to collapse an end to end nested Monad Stack into a single type 👌.

![Transformers YT](https://www.youtube.com/watch?v=kE73sSTzZzM)

Impressive huh? That’s exactly what you get when you **collapse your Monad Stack into a single type**. So I mean, why wouldn’t you do it?

If you haven’t read [the first part on this series](https://medium.com/@JorgeCastilloPr/kotlin-functional-programming-i-monad-stack-518d1bd8fbee), please do it before reading this. It will help you to understand. Also, this post is a further iteration on that approach.

---

### The problem

Something very well said on the FP world is that **Monads do not compose**. At least not gracefully. They just compose when you return the same monad on all the functions on a call chain, so you can use `flatMap` or `flatten` to collapse the stack. But that’s a very unlikely situation.

In a standard scenario, your functions would probably return different monads at different levels, and your stack could end up being deeply nested.

When you get many nested levels on the stack, it starts looking like a pyramid. Check out this snippet where you have 3 levels of nested monads. Here we just want to get access to the innermost level:

```kotlin
data class Country(val code: String)
data class Address(val id: Int, val country: Option<Country>)
data class Person(val id: Int, val name: String, val address: Option<Address>)

fun getCountryCode(maybePerson : Either<BizError, Person>): Either<BizError, String> =
  maybePerson.flatMap { person ->
    person.address.toEither({ AddressNotFound(person.id) }).flatMap { address ->
      address.country.fold({ CountryNotFound(address.id).left() }, { it.code.right() })
    }
  }
```

This approach can exponentially lead to scary situations.

![hadouken](assets/images/hadouken.jpeg)

### Monad comprehensions

On the previous snippet we were playing with Either at all levels. It’s a single type, so we can apply something cool called **Monad Comprehensions** for the sugar!

If you know a bit about Haskell you have probably heard about something called *“do notation”*. It’s a way to write sequential code that could be “flatMapped” with a well aligned sequence of operations declared one after another. You can also “yield” a result in the end. This is just for syntax.

[Arrow](https://arrow-kt.io) provides that under the monad context using something called bindings. They enable monad comprehensions for all datatypes for which a monad instance is available.

So using bindings the previous snippet could also be written like:

```kotlin
fun getCountryCode(maybePerson : Either<BizError, Person>): Either<BizError, String> =
  Either.monadError<BizError>().binding {
    val person = maybePerson.bind()
    val address = person.address.toEither({ AddressNotFound(person.id) }).bind()
    val country = address.country.toEither({ CountryNotFound(address.id) }).bind()
    yields(country.code)
  }.ev()
```

Bindings are run over a monad context, for example a `MonadError` instance. Binding blocks can contain any number of sequential “flatMappeable” operations, declared one after another. Each operation result is automatically lifted to the monad context and “flatmapped” to the next line thanks to the `bind()` call.

In the end you can `yield` a result which will also automatically be lifted to the context of the monad being used.

This syntactic sugar feature can be found on many languages. Every time you have a bunch of sequential operations that return the same monad type and provide a final result lifted to the same context, that’s prone to use a binding.

But remember: we can do this here mainly because we are just playing with a single type: `Either`. What happens when we have different nested types?

> This example is part of one of the [Arrow documented datatypes: EitherT](https://arrow-kt.io/docs/arrow/data/eithert/). Arrow docs are very didactic. They provide detailed samples with code snippets and further explanations.

### Real world problem

You also have a deep example of this situation on the previous post, where we built our monad stack step by step to solve the main concerns for any Android app. Then we ended up having the following stack:

```kotlin
Reader<GetHeroesContext, IO<Either<CharacterError, List<SuperHero>>>
```

So there we had our composed `Reader` computation expecting a context providing the required dependencies to run. When dependencies are provided, unsafe effects wrapped on `IO` are performed, which returnEither an error or a valid list of heroes.

The stack was starting to get a bit complex, therefore **readability decreased**.

But the monad stack approach is actually incomplete. Functional developers wouldn’t stay there but iterate further, and use **Monad Transformers** to solve the issue.

### So what’s a Monad Transformer?

It’s similar to a regular monad, but it’s not a standalone entity: instead, it modifies the behaviour of an underlying monad.

Most of the monads provided by [Arrow](https://arrow-kt.io/) have transformer equivalents.

By convention, the transformer version of a monad has the same name followed by a `T` . For example, the transformer equivalent of `State` is `StateT`.

When you need an already existing monad like `IO<A>` to obtain the capabilities of a different monad like `Either<L, R>`, you use a transformer.

### How do we benefit from it?

By using transformers you **remove the need for nested types and collapse all of them into a single one** that has the same capabilities than all the previous ones by itself.

That means you get rid of the complexity provoked by nested monads, where you are forced to `map` / `flatMap` over each level to gain access to the next one in the stack. That can become unusable very rapidly on deep stacks.

So your code can become quite simpler thanks to Transformers.

![transformer gif](assets/images/transformer.gif)

*Our deadly machine will be ready soon!*

### Fixing the stack

Let’s pick the deepest nested types, which would be the following:

```kotlin
IO<Either<CharacterError, List<SuperHero>>>
```

We have an `IO` computation wrapping a side effect. To gift it with the duality powers of `Either`, we can use **Either Transformer**.

It’s type is: `EitherT<F, L, A>`, where `F` is the underlying monad type, `L` is the left type for the `Either` and `A` is the gonna be the final resulting type.

For our super heroes example it the types would be:

```kotlin
EitherT<IOHK, CharacterError, List<SuperHero>>
```

Here, you need to understand that `IOHK` is the way we use to refer to IO on a generic type slot. So IO is here the underlying monad.

It’s part of the boilerplate automatically generated at compile time for all the higher kinds by Arrow. We will describe higher kinds and their benefits on further posts so let’s obviate it for now.

Thanks to `EitherT`, `IO` is now capable of retaining typed errors (not just throwables) 🎉. But the stack is a bit bigger:

```kotlin
Reader<GetHeroesContext, IO<Either<CharacterError, List<SuperHero>>>
```

So we need to gift the powers of `Reader` to the already transformed `IO`, so it can also provide dependency injection and not just `IO` deferred computations and error handling, which is what we have for the time being.

We can use the **Reader Transformer** to achieve that: `ReaderT<F, D, A>`.

`F` is again the underlying monad being transformed, `D` will be the `Reader` context with all the dependencies, and `A` will be the final result type.

The transformed type requires to be agnostic of what’s `A`, since we want it to be able to work over different return types. In the other hand, I am fixing the left type of `Either` as `CharacterError` since I know that any possible error on my system is going to be mapped to one of the domain `CharacterError` sealed class implementations. (Check [the previous post](https://jorgecastilloprz.github.io/kotlin-fp-1-monad-stack) for more details).

This time, the underlying monad being transformed is going to be: `EitherT<IOHK, CharacterError, List<SuperHero>>`. We need to fit that on the `F` position of our transformer type `ReaderT<F, D, A>`.

That means we must partially apply `EitherT` to fix the `IOHK` and `CharacterError` types, so we just leave `A` as generic. Remember that its type is `EitherT<F, L, A>`.

```kotlin
ReaderT<EitherTKindPartial<IOHK, CharacterError>, D, A>
```

And with this we would have our transformed type ready to operate. Even if it can still look like nested types, it is indeed a single one with all the required capabilities.

It’s also important to notice that the `ReaderT` is also called **`Kleisli`**. It’s indeed a type alias in Arrow.

```kotlin
typealias ReaderT<F, D, A>  = Kleisli<F, D, A>
```

![fuse](assets/images/fuse.gif)

`Kleisli<F, D, A>` is just a wrapper around the function `D => F<A>`. That’s exactly what we have on each one of our computation levels or layers on the sample repo architecture.

You will also notice that I’m aliasing the transformed type to `Result` to improve its semantics, since at the end of the day it’s gonna be representing an action result moving across the different layers.

```kotlin
typealias Result<D, A> = Kleisli<EitherTKindPartial<IOHK, CharacterError>, D, A>
```

Im also wrapping `Result` into a datatype called `AsyncResult` that delegates all its functions to the `Kleisli` but at the same time provides handy monad instances to compose `AsyncResults` and use bindings over them. This is going to be really helpful when it comes to the architecture implementation.

So let’s take a look on how the complete architecture of the app is built thanks to the `AsyncResult` type.

### Architecture using AsyncResult

We are going to start on the outermost layer of Clean, where the frameworks would be. There you would have your data sources and view implementations. Let’s start having a look at the network data source implementation.

```kotlin
fun <D : SuperHeroesContext> fetchAllHeroes(): AsyncResult<D, List<CharacterDto>> =
    AsyncResult.monadError<D>().binding {
      val query = buildFetchHeroesQuery()
      val ctx = AsyncResult.ask<D>().bind()
      runInAsyncContext(
          f = { fetchHeroes(ctx, query) },
          onError = { liftError<D>(it) },
          onSuccess = { liftSuccess(it) },
          AC = ctx.threading<D>()
      ).bind()
    }.ev()
```

We have a collapsed transformed type thanks to `AsyncResult`, so we can just wake up a `binding` over it, and declare a bunch of sequential operations inside.

Here I am composing an operation by all those sequential ones. **I don’t really run the composed operation at all but return it so I can keep composing on top of it** thanks to flatMap, map and so on. That’s key and it’s enabled thanks to transformers.

The sequential operation blocks inside my binding that will be run when we decide it’s time to run the whole function chain, are:

* Build the super heroes fetching query
* Porvide access to the dependencies by lifting a `Reader` (which would be `AsyncResult` by itself thanks to transformers) and getting access to the reader context.
* Running the IO computation.

Each one of those operations can optionally call `bind()` in the end to force resolution and bind the result to the context of the monad being used, which is an `AsyncResultMonadError` instance. That forces the next step to wait for resolution on the previous one so the resulting monad from it is going to be available to compute.

For more details on the `runOnAsyncContext` function please take a look at [the previous post](https://jorgecastilloprz.github.io/kotlin-fp-1-monad-stack). It’s mainly going return an `IO` computation that runs the `f` lambda code into a coroutine and provides lambdas for both success and error cases.

Then I have a kind of stub repository layer where you would be supposed to implement cache policies and so on.

```kotlin
fun <D : SuperHeroesContext> getHeroes(policy: CachePolicy): AsyncResult<D, List<CharacterDto>> = when (policy) {
  is NetworkOnly -> fetchAllHeroes()
  is NetworkFirst -> fetchAllHeroes() // TODO change to conditional call
  is LocalOnly -> fetchAllHeroes() // TODO change to local only cache call
  is LocalFirst -> fetchAllHeroes() // TODO change to conditional call
}
```

I wouldn’t personally add this function to the call chain if I don’t have any further logic to add, obviously, but it’s there just for didactic purposes on comparing this approach with an OOP based Clean Architecture one.

You could easily apply operations here to implement the cache policy strategies like flatMapping or mapping over the composed computation returned from the data source.

> Please notice that all the functions I am showing here are defined at a package level, I am not playing with useless enclosing instances at all since it’s FP and all the dependencies are passed in as function arguments or provided by the reader. There is no need to have enclosing instances containing this functions, since there is no state to keep on them. IMO, It’s key to understand that. OOP and FP designs are highly different.

The caller for the repo would be the Use Case most likely, where the business logic would be.

```kotlin
fun <D : SuperHeroesContext> getHeroesUseCase(): AsyncResult<D, List<CharacterDto>> =
    getHeroes<D>(NetworkOnly).map { discardNonValidHeroes(it) }
```

I just want to filter out some “non valid heroes” here so I can map over the data source returned computation and do it. Thanks to the fact that we are returning AsyncResults on all levels we can easily compose on top of it since it’s now a single type.

Presentation code could look like this:

```kotlin
// All this methods are defined at a package level on a presentation.kt file.
fun getSuperHeroes(): AsyncResult<GetHeroesContext, Unit> =
    getHeroesUseCase<GetHeroesContext>()
        .map { charactersToHeroes(it) }
        .flatMap { drawHeroes(it) }
        .handleErrorWith { displayGetHeroesErrors(it) }

fun charactersToHeroes(characters: List<CharacterDto>): List<SuperHeroViewModel> =
    characters.map(::characterToHero)

fun drawHeroes(heroes: List<SuperHeroViewModel>): AsyncResult<GetHeroesContext, Unit> =
    AsyncResult.monad<GetHeroesContext>().binding {
      val ctx = AsyncResult.ask<GetHeroesContext>().bind()
      ctx.view.drawHeroes(heroes)
      AsyncResult.unit<GetHeroesContext>()
    }.ev()

fun displayGetHeroesErrors(c: CharacterError): AsyncResult<GetHeroesContext, Unit> =
    AsyncResult.monad<GetHeroesContext>().binding {
      val ctx = AsyncResult.ask<GetHeroesContext>().bind()
      when (c) {
        is NotFoundError -> ctx.view.showNotFoundError()
        is UnknownServerError -> ctx.view.showGenericError()
        is AuthenticationError -> ctx.view.showAuthenticationError()
      }
      AsyncResult.pure<GetHeroesContext, Unit>(Unit)
    }.ev()
```

I can map or flatMap over the Use Case result to apply mapping and effects, and even handle errors using the `handleErrorsWith` function to apply error related effects on the view.

The `getSuperHeroes()` and both side effect causing functions `drawHeroes()` and `displayGetHeroesError()` would have a return type like the following: `AsyncResult<GetHeroesContext, Unit>`, since the moment those get finally unfolded and their effects are run, they would be applying just side effects on the view.

So nothing is returned from an action like that. Therefore the `Unit` type. And of course the `GetHeroesContext` is just the type we are fixing here for the reader context, where the dependency bindings are defined.

Take a look on both side effect applying functions which are also using the power of bindings to do their work!

```kotlin
fun drawHeroes(heroes: List<SuperHeroViewModel>): AsyncResult<GetHeroesContext, Unit> =
    AsyncResult.monad<GetHeroesContext>().binding {
      val ctx = AsyncResult.ask<GetHeroesContext>().bind()
      ctx.view.drawHeroes(heroes)
      AsyncResult.unit<GetHeroesContext>()
    }.ev()

fun displayGetHeroesErrors(c: CharacterError): AsyncResult<GetHeroesContext, Unit> =
    AsyncResult.monad<GetHeroesContext>().binding {
      val ctx = AsyncResult.ask<GetHeroesContext>().bind()
      when (c) {
        is NotFoundError -> ctx.view.showNotFoundError()
        is UnknownServerError -> ctx.view.showGenericError()
        is AuthenticationError -> ctx.view.showAuthenticationError()
      }
      AsyncResult.pure<GetHeroesContext, Unit>(Unit)
    }.ev()
```

I’m mainly using bindings here to get access to the Reader context where they view contract is defined as a dependency, and then apply effects on the view using it. So if I had to apply many different effects on the view I could just write one after another on this way.

Again a cool point is that all those computations are built on top of the UseCase already composed result. Which means effects are **still not run**. We keep deferring them until the view implementation decides it’s the right moment to run and unfold the whole call chain, finally performing all the computations and effects!

```kotlin
override fun onResume() {
    super.onResume()
    getSuperHeroes().unsafePerformEffects(heroesContext)
  }

// extension function just for some usage syntax
fun <D : SuperHeroesContext, A> AsyncResult<D, A>.unsafePerformEffects(ctx: D) {
  this.run(ctx).value.ev().unsafeRunAsync {}
}
```

The view implementation would decide when to finally run the returned composed computation by the presentation layer. Some complexity related to how we are emulating Higher Kinds is still visible there (like the evidence function ev(), but hopefully [KEEP-87](https://github.com/Kotlin/KEEP/pull/87) will be eventually successful!. Don’t forget to upvote it :). Otherwise we will just keep the HK and Typeclasses emulation and keep iterating over it to make it easier to use.

### What about testing?

Testing becomes trivial when you have a reader in place. In our case a reader transformer. You just need to provide a different context implementation that can be providing mocks for the required dependencies.

So you can keep exercising the real production code base and just pass a different context to assert over the whole system execution. Black box tests are quite ese on this scenario.

So let’s take a rapid look on how I am declaring the Reader context and we will be done. Promised!

```kotlin
sealed class SuperHeroesContext {

  lateinit open var ctx: Context

  open val heroDetailsPage = HeroDetailsPage()
  open val apiClient
    get() = CharacterApiClient(Builder(
        BuildConfig.MARVEL_PUBLIC_KEY,
        BuildConfig.MARVEL_PRIVATE_KEY).debug().build())

  open fun <D: SuperHeroesContext> threading() = AsyncResult.asyncContext<D>()

  data class ApplicationContext(override var ctx: Context) : SuperHeroesContext()
  data class GetHeroesContext(override var ctx: Context, val view: SuperHeroesListView) : SuperHeroesContext()
  data class GetHeroDetailsContext(override var ctx: Context, val view: SuperHeroDetailView) : SuperHeroesContext()
}
```

This is just a way to build it, where I am kind of mimicking what we are using to build in Android with Dagger scopes per Application and Activity.

In the end it can be a sealed hierarchy that can be implemented in different ways depending on the scope. This approach helps us to define overridable global dependencies like the Android `Context`, navigators, api clients or threading implementations. Those would be application scope dependencies most probably.

But all of them are completely overridable by any of the “sub-scopes” declared below. So you can easily add your own sealed class implementation for providing mocks by overriding properties or methods from the sealed class 🎉.

For activity scopes, you can also override context with the current activity one and provide a different instance for the view contract. You can easily provide all this since graphs / components are always created from the view implementation, the same way we do with Dagger.

---

### Wrapping up

Remember that you can [add me on Twitter](https://www.twitter.com/JorgeCastilloPr) to know more about this topic and many more.

And take a look [to the previous post of the series if you still didn’t](https://jorgecastilloprz.github.io/kotlin-fp-1-monad-stack)!

![Optimus Prime](assets/images/transformer.jpeg)

*Optimus prime approves this post.*
