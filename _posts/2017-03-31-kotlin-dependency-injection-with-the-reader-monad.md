---
layout: post
current: post
cover: assets/images/copenhaguen.jpeg
navigation: True
title: Kotlin Dependency Injection with the Reader Monad
date: 2017-03-31 10:18:00
tags: [kotlin, fp]
class: post-template
subclass: 'post'
author: jorge
---

In this post you will learn how to provide seamless dependency injection to your Functional Programming architecture using the `Reader` Monad. Leave Dagger, Koin and Kodein behind!

As an Android dev I am used to invest a lot of time on coding infrastructure to provide *Dependency Injection* to my apps. But guess what: **Dependency Injection is not Dagger, neither Koin or Kodein**.

### Main differentiation

In purpose of avoiding to repeat the term *Dependency Injection* all over the place, let me just refer to it as *DI* from now on.

When we talk about *DI*, we are not referring to any concrete framework. *DI* is a very simple concept, and means **to get collaborators of a class passed in by using constructor, simple setters, or alternatively more advanced techniques like compile time code generation** to create the code to provide those instances at runtime. (Some people thinks *DI* is just the magic frameworks like `Dagger` or `Spring` do).

If we step back and look at the concept in a more abstract way, the only thing to remember is that we gain the possibility to provide collaborators to any of our entities at runtime. Now we can provide different implementations of those collaborators by using extension (polymorphism), interface implementation, or whatever technique we want to use. By doing that, we are **changing our class’s behavior**.

Of course I'm thinking about testing here.

So DI is a concept, and we can implement it in many different ways.

### Why to switch our approach

I am not saying you must leave frameworks like *Dagger*, *Spring*, *Kodein*, *Koin*, or whatever behind. That’s your choice, and of course you can keep using them, if those fit your team & your code base. Nothing prevents you from keeping those forever, or may be from making a gradual port over time to a different *DI* system. Actually, I'd suggest you to be thoughful on these sort of decisions that end up affecting your whole team.

The only intention of this post is to open your mind a bit more about the concept, and maybe look for different approaches instead of just going straight for the only one you might know the next time you need to implement *DI*.

### About Functors, Applicatives, and Monads

I don’t really want to go so deep on what `Functors`, `Applicatives` and `Monads` are for this blogpost. If you want to know more, I would recommend you to read [this article series](https://hackernoon.com/kotlin-functors-applicatives-and-monads-in-pictures-part-1-3-c47a1b1ce251) translated to Kotlin by [Alberto Ballano](https://twitter.com/Aballano), or [the original one written for Haskell](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html). Both are pretty good.

But don’t worry, you don’t really need to understand those concepts in a very detailed way to work with the `Reader`.

To simplify things, lets just think about the three mentioned types as structures capable of wrapping an element which also provide some useful combinators to help you work with it in a functional style. That element could be a value, **or even a function**.

### Reader

The `Reader` is a `Monad` that wraps a **function** that is not going to be run at this very moment but at some point in the future. We store the function into the `Reader` to defer its execution to the moment when we can provide a proper **execution context** for it.

And that's the key here. `Reader` wraps a function that **requires some context (dependencies) to run**. (💡 it can assume that whenever the deferred function runs, the dependencies will for sure be there, otherwise it wouldn't be running!).

To ask the reader to run the function when the context is ready, we would do something like this:

```kotlin
myReader.run(ctx)
```

Where `ctx` is going to be the reader context, which can be a class containing all the required dependencies for the `Reader`'s function.

The code in the snippet will run the function inside of the `Reader`, implicitly passing the `context` to it as it’s input parameter.

Using this approach, we could code a complete data flow without the need to worry about passing our dependencies all the way down. Dependencies (context) will be instantiated and passed in from the edge of our system, in the moment we proactively run the method chain. Beyond that point, `Readers` will take care of implicitly passing those dependencies deep into the architecture.

To achieve that, we can concatenate `Readers` across different layers and run the whole chain when the dependency tree resolution can be provided.

So cheers for us, because with this approach we are going to be able to provide `DI`. **BOOM!** 💥

![ari_gold](assets/images/ari_gold.gif)


> The benefit of using `Readers` is to be able to provide an execution context in an implicit way without the need to state it explicitly on each one of the methods involved.

### Implementation

They might say that *one code snippet is better than a thousand words*. Or I might be the only one saying it, but here you have a simple version of the `Reader` written in `Kotlin` (not the actual one we're using in Arrow but a simplified one for the sake of the example):

```kotlin
class Reader<D, out A>(val run: (D) -> A) {

  inline fun <B> map(fa: (A) -> B): Reader<D, B> = Reader {
    d -> fa(run(d))
  }

  inline fun <B> flatMap(fa: (A) -> Reader<D, B>): Reader<D, B> = Reader {
    d -> fa(run(d)).run(d)
  }

  companion object Factory {
    fun <D, A> just(a: A): Reader<D, A> = Reader { _ -> a }

    fun <D> ask(): Reader<D, D> = Reader { it }
  }
}
```

If you are not familiarized with functional structures, you have different things to look at here before moving on:

* Mainly, the class works with types `D` and `A`. `D` stands for the reader context, and `A` is going to be the result type for the deferred function.
* On construction, we need to provide a function of type `f: (D) -> A` , which is a function capable of getting the context as an argument and providing a result of type `A`. So this is going to be the deferred function stored within the `Reader`.
* There is a **`map`** combinator which receives a `fa: (A) -> B` as an argument to transform an `A` value to a `B` value. Standard `map` combinator right? It transforms the inner value applying the given function to it. In this case it'll apply `f` to the already wrapped function in the `Reader`. So it effectively composes both functions.
* **`flatMap`** is not highly different. In this case, the mapping function passed in has the type `fa: (A) -> Reader<D, B>`. If we passed that function to the **`map`** combinator, we would end up having a nested `Reader` as a result, like `Reader<D, Reader<D, B>>`. Since we want to get a flattened result, **`flatMap`** implementation is prepared to return a simple `Reader<B>`. Note that **`flatMap`** can help us on `Reader` concatenation.
* The `companion object` includes a **`just`** method to lift a `Reader` from an `A` resulting value, and also an **`ask`** method to create a `Reader` just from the context (it wraps a function with type `(D) -> D`).

Of course the Arrow reader has more useful combinators, but I am intentionally keeping this as simple as I can to ease the path for newcomers.

### Show me the code 🙏🏽

Of course! Let’s see some indications about how to use the Reader in our programs written in `Kotlin`.

Let's think about the typical application using a decoupled architecture divided in layers, like `Clean`. Let’s say we are using the `MVP` pattern to separate the view from the business logic. With that in mind, the execution flow to retrieve some data could be like this:

```
view event -> presenter -> use case -> repository -> datasource
```

So the `view` (let’s use a simple activity or a fragment) could wake up the first `Reader` in the execution chain by doing:

```kotlin
class MyView : View {
	// ...
	override fun onResume() {
	  super.onResume()
	  presenter.getSuperHeroes().run(
		    GetHeroesContext(
		        view = this,
		        getSuperHeroesUseCase = GetSuperHeroesUseCase(),
		        heroesRepository = HeroesRepository(),
		        dataSources = listOf(MemoryHeroesDataSource())
		    )
		)
	}
}
```

That’s possible because `presenter.getSuperHeroes()` returns a `Reader`.

Also **note that we're passing all our dependencies from here, the "edge of the world", or in other words, the entry point to our system.** Here's where we can perform our side effects and build our dependency graphs.

And this is how the presentation layer function looks like:

```kotlin
fun getSuperHeroes(): Reader<GetHeroesContext, Unit> =
        ReaderApi.ask<GetHeroesContext>().flatMap { ctx ->
            ctx.getSuperHeroesUseCase.getSuperHeroes().map {
                if (it.isEmpty()) ctx.view.showHeroesNotFoundError()
                else ctx.view.drawHeroes(it.map { SuperHeroViewState(it.name) })
            }
        }
```

We can use **`ask`** to get a `Reader<D, D>` just from the context, so the function it wraps has type `(D) -> D` which means it asumes it'll get a context (`D`) at runtime and will just return it.

Then we can **`flatMap`** over it to get access to its `context`. Let's say this is how our context (aka: program dependencies) looks:

```kotlin
data class GetHeroesContext(
    val view: View,
    val getSuperHeroesUseCase: GetSuperHeroesUseCase,
    val heroesRepository: HeroesRepository,
    val dataSources: List<HeroDataSource>
)
```

It's a simple data class containing our dependency graph. Our graphs could be more complex and even composed in a real world scenario, this is for the sake of the example.

So, since we flatmapped over the `Reader<D, D>` we got access to its context (`D`) and we can use it to get the reference to the `use case`.

We also used **`map`** combinator over the use case result to apply the side effects and notify the view for rendering results or displaying errors in case there was a problem.

So, overall, and thinking about the execution flow, this is what's happening:

* The main `Reader` receives the context, and in response it’s going to call `flatMap`.
* `flatMap` will have as input another function that receives the context and returns a mapped `Reader` for side effects over the view. This mapped `Reader` will be run automatically when you pass the context to the outer one, since both are chained by `flatMap`.

Here you have the `use case` method implementation:

```kotlin
fun getSuperHeroes(): Reader<GetHeroesContext, List<SuperHero>> =
	ReaderApi.ask<GetHeroesContext>().flatMap { ctx ->
	    ctx.heroesRepository.getHeroes()
	}
```

This one is much easier, but follows the same principle. Again, We use **`ask`** to **`flatMap`** and get access to the `ctx`. Then we are able to take out the dependency required from it and run the following layer's computation. In this case, it’s going to be the repository. So we are forwarding the call when the context arrives, and we are also flattening the `Reader` returned by the repo.

This is kind of non too realistic, as it’s not adding any business logic value to the result. If you needed to operate over the repository result to apply some domain logic over it, you could do the same we are doing with the **`map`** method for the presenter function.

Here's the repository logic:

```kotlin
fun getHeroes() = ReaderApi.ask<GetHeroesContext>().flatMap { ctx ->
    ctx.dataSources[0].getAll()
}
```

Same reasoning one more time. Normally I probably wouldn’t even have a repository if we are not coordinating different data sources, but I want to follow all the `Clean Architecture` “standards” for the sample project. Here you could have logics related to caching policies, composing domain models returned from different data sources, or similar things.

And finally we could have a `DataSource` logic like the following one:

```kotlin
override fun getAll() = ReaderApi.ask<GetHeroesContext>().map {
        listOf(
            SuperHero("IronMan"), SuperHero("Spider-Man"),
            SuperHero("Batman"), SuperHero("Goku"), SuperHero("Vegeta"), SuperHero("SuperMan"),
            SuperHero("Ant-Man"), SuperHero("Krilin"), SuperHero("Super Mario"),
            SuperHero("Wolverine"), SuperHero("Massacre"), SuperHero("Jake Wharton"),
            SuperHero("Jesus Christ"), SuperHero("Donald Trump (villain)")
        )
    }
```

So this one would complete the execution chain, and we should be ready to go! 🎉

### So what do we get from this approach ?

* You have a different way to inject dependencies in your system without the need of complex external frameworks.
* You get a good synergy between the way you inject dependencies and the way you code your app logic. Everything is done in a more functional style, so it’s better adapted to a modern lang like Kotlin with support for high order functions.
* **You avoid passing dependencies as function arguments at every single level in your architecture**, as those are hidden inside of the Reader's context, so you get collaborators injected implicitly by the `Reader`.

---

Please, feel free to [follow me on Twitter](https://www.twitter.com/JorgeCastilloPr) to discuss anything related (or not even related!) to this article. I usually post and retweet about `Kotlin` and any other Android development and functional related posts.

In further posts I we will review how to raise the level of abstraction over the `Reader` monad using something we call the "`Kleisli`" so it can work over different data types. We will also start reaching the real marvel API to fetch some amazing **real super heroes like Goku**. (Yes, he's the only real super hero).

Stay tunned!

![goku](assets/images/goku.gif)
