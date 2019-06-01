---
layout: post
current: post
cover: assets/images/copenhaguen.jpeg
navigation: True
title: Kotlin Dependency Injection with the Reader Monad
date: 2017-03-31 10:18:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

As an Android developer I am used to waste a lot of time on creating required infrastructure to provide *Dependency Injection* to my apps. I normally use `Dagger`, which is the most known framework out there, and I guess you all do the same. But guess what: **Dependency Injection is not Dagger**. Kind of obvious, right ? But many people is confused about it.

## Main differentiation

In purpose of avoiding to repeat the term *Dependency Injection* all over the place, let me just refer to it as *DI* from now on.

When we talk about *DI*, we are not referring to any concrete framework. *DI* is a very simple concept, and means to get collaborators of a class passed in by using constructor, simple setters, or even more advanced techniques like compile time code generation to create the code to provide those instances at runtime. But the approach you choose lacks importance.

If we step back and look at the concept in a more abstract way, the only thing to remember is that we gain the possibility to provide collaborators to any of our entities any point in time. Now we can provide different implementations of those collaborators by using extension (polymorphism), interface implementation, or whatever technique we want to use. By doing that, we are changing our class’s behavior.

Of course we are thinking about testing.

So DI is a concept, and we can implement it in different ways.

## Why to switch our approach

I am not going to say that you can’t keep using *Dagger* even if you are moving your apps to *Kotlin*. That’s your only choice, and of course you can. Nothing prevents you from keeping it forever, or may be from making a gradual port over time to a different *DI* system. You might just keep using *Java* for those classes, or may be switch to *Kotlin* by adding some `lateinit` modifiers to injected fields here and there.

The main intention of this post is to open your mind a bit more about the concept, and maybe look for different approaches instead of just going straight for the only one you might know the next time you need to implement *DI*.

## About Functors, Applicatives, and Monads

I don’t really want to go so deep on what `Functors`, `Applicatives` and `Monads` are for this blogpost. If you want to know more, I would recommend you to read [this article series](https://hackernoon.com/kotlin-functors-applicatives-and-monads-in-pictures-part-1-3-c47a1b1ce251) translated to Kotlin by [Alberto Ballano](https://twitter.com/Aballano), or [the original one written for Haskell](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html). Both are pretty good.

But don’t worry, you don’t really need to understand those concepts in a very detailed way to work with the `Reader`. I am not an expert in `Monads` either, I must say.

To simplify things a lot, lets just think about the three mentioned types as structures capable of wrapping an element which also provide some useful combinators to help you work with it in a sort of functional style. That element could be a value, **or even a function**.

## Reader

The `Reader` would be a `Monad` containing a **function** that is not going to be run at this very moment but at some point in the future. We store the function into the `Reader` to defer its execution to the moment when we can provide a proper execution context for it.

To ask the reader to run the function when the context is ready, we would do something like this:

```kotlin
myReader.run(c)
```

Where c is going to be the Context, which is a class containing all the needed instances to resolve the Reader's function execution. So the previous line will run the function inside of the Reader, internally passing the context to it as it’s input parameter.

Using this approach, we could code a full flow of data without the need to worry about creating any instance. Those will be instantiated and passed in from the edge of our system, in the moment we need to run the method chain. In other words, we can concatenate methods returning Readers across different layers and run the whole chain when the dependency tree resolution can be provided.

So cheers for us, because with this approach we are going to be able to provide DI. BOOM! 💥


Ari Gold just found out he can inject dependencies using the Reader
Now you will probably go crazy thinking that it’s not really different from passing the execution context to all the methods involved in the data flow chain from the starting point to the end, one after another 😅. And you might be correct just in part.

The real benefit of using Readers is to be able to provide that context in an implicit way without the need to state it explicitly on each one of the methods involved.

Implementation
They might say that one code snippet is better than a thousand words. Or I might be the only one saying it, but here you have a simple version of the Reader written in Kotlin:


If you are not familiarized with functional structures, you have different things to look at here before moving on:

Mainly, the class works with types C and A. C is going to represent the reader context used, and A is going to be the result type for the deferred function.
On construction, we need to provide a function of type f: (C) -> A , which is a function capable of getting the context as an argument and providing a result of type A. So this is going to be the deferred function stored inside of the Reader.
There is a map combinator which receives a fa: (A) -> B as an argument to transform an A value to a B value. According to standard map convention, map returns a new monad of the same type (Reader in this case), containing the mapped element of type B.
flatMap is not highly different. In this case, the mapping function passed in has the type fa: (A) -> Reader<C, B> . If we passed that function to the map combinator, we would end up having a duplicated reader as a result, like Reader<C, Reader<C, B>> . Since we want to get a flattened result, flatMap implementation is prepared to return a simple Reader<B>. flatMap can help us on Reader concatenation.
zip combinator is handy to combine two Readers into a single one that will contain a function capable of receiving the context c, and returning a Pair<a, b>.
local can be used to combine Readers with different context scopes.
The companion object includes a pure method to lift a Reader from an A value, and an ask method to be able to flatMap over the context.
Of course, we could add plenty of additional useful combinators, but I am intentionally keeping this as much simple as I can to ease the path for newbies like me.

Could you please stop the bullshit and just paste some code ?
Of course I can. Let’s see some indications about how to use the Reader in our Android apps written in Kotlin.

Just think about the typical app using a decoupled architecture divided in layers, like Clean. Let’s say it’s also using the MVP pattern to separate the view from the business logic. With that in mind, the execution flow to retrieve some data could be like this, in both directions:

view event -> presenter -> use case -> repository -> datasource
So the view (let’s use a simple activity) could wake up the first Reader in the execution chain by doing:


That’s possible because presenter.getSuperHeroes() returns a Reader. And this is how the presentation layer function looks like:


Here we have kind of a minor trick to clarify. We coded an ask method for the Reader inside of its companion object, so you can flatMap over the context itself without the need for a Reader instance. That gives us access to the context and it’s properties, so we can get the reference to the use case(aka Interactor) and run it.

You can have more details about how ask is implemented by browsing the sample repository. You will find out that it returns a new Reader with the context working as the dependency and the value at the same time.

We also use the map combinator over the use case result to apply the side effects and notify the view for rendering results or displaying an error.

So, overall, and thinking about the execution flow, this will happen:

The main Reader will receive the context, and in response it’s going to call flatMap .
flatMap will have as input another function that receives the context and returns a mapped Reader for side effects over the view. This mapped Reader will be run automatically when you pass the context to the outer one.
But if you look carefully, you will see that this Reader is mapping over an inner one returned by interactor.getSuperHeroes(). So when the one inside the flatMap receives the context, it would first run this inner Reader and then map over it’s result.
Here you have the use case method implementation:


This one is much easier, but follows the same principle. Again, We use ask to flatMap over the ctx and run the following layer function. In this case, it’s going to be the repository. So we are forwarding the call when the context arrives, and we are also flattening the Reader returned by the repo.

This is kind of non too realistic, as it’s not adding any business logic value to the result. If you needed to operate over the repository result to apply some domain logic over it, you could do the same we are doing with the map method for the presenter function.


Here you have another forwarding one. Normally I probably wouldn’t even have a repository if we are not coordinating different data sources, but I want to follow all the Clean Architecture “standards” for the sample project.

You might have noticed that I am not using ask to forward this call. That’s because the datasource does not return a Reader that we need to flatten, but just a plain list of heroes. Here you have it:


So this one would end up the execution chain, and we should be ready to go.

So what do we get from this approach ?
You have a different way to inject dependencies in your system without the need of complex external frameworks.
You get a good synergy between the way you inject dependencies and the way you code your app logic. Everything is done in a more functional style, so it’s better adapted to a modern lang like Kotlin with support for high order functions.
You avoid passing dependencies as function arguments all over the place, as those are hidden inside of the Reader's context, so you get collaborators injected in some sort of implicit way.
You don’t need to fight to Dagger 2 validation errors and the tediously weird boilerplate you need to add for component configurations all the time.
Don’t forget to take a look at the sample repository to know more details about the implementation. The repo is kind of a playground, so don’t feel weird if you find that it evolves too fast or you find some dead code in some parts of it.

Please, feel free to add me on Twitter @jorgecastillopr to discuss anything related (or not even related!) to this article. I usually post and retweet about Kotlin and any other Android development and functional related posts.

In the following weeks I will publish another article that will help us to raise the level of abstraction over the Reader monad and over the way we work with it. We will also start reaching the real marvel API to fetch some amazing real super heroes!

Stay tuned!


You too know Goku is the only real super hero.
