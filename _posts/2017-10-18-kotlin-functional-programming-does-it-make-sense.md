---
layout: post
current: post
cover: assets/images/flowerfield.jpeg
navigation: True
title: Kotlin Functional Programming: Does it make sense?
date: 2017-10-18 15:18:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

Functional programming can feel like a big wave of fresh air over a beautiful landscape at sunset. The flowers are literally us. 😍 🌼 (maybe?)

So no jokes: it’s clearly worth it!

### Is Kotlin a FP lang?

When we think about Kotlin, OOP is always implicit. But that’s not strictly necessary. Kotlin is still **open to a different paradigm** thanks to some of its built-in features. I’m obviously referring to FP.

If you take a careful look at the built-in features, you will find some very obvious ones like *high order functions or the power to use functions as first class citizens*. Those features are clearly related to **function oriented styles**, even if that does not strictly mean Functional Programming. But it opens a whole new world of approaches not possible before in front of you. (I assume you are an Android dev like me, so you have been using Java for a big while).

Having the possibility to pass functions as function arguments, or return functions from other functions opens the language to new techniques like **deferred computation**. If you return functions on your different layers instead of already computed values, what you get back from it is a composed deferred computation, or in other words, **lazy evaluation**. Your computations are deferred in time, still not run.

This is completely related to FP, and you will see why in the following lines.

Also *functions as first class citizens*, as you already know, means you can just use functions the same way and on the same places you where using classes on Java. So, you are able to store functions on variables, for example, or even declare them at a package level. Because honestly, do we always need to play with instances all the time?, or is it kind of “imposed” by our paradigm and the libraries we use?

You can also stare at the language a bit longer, and you will be able to find tons of additional features that you can find on many type safe FP languages: Type inference, operator overloading, good support for syntax improvements (implemented by *extension functions*), collection operations as `map`, `flatMap`, `fold`, `reduce`…, or even algebraic data types (`sealed` classes).

The language is even abstracting some logics and behaviors to functions, and not to classes, as you would do in OOP. That’s something very usual on **FP vs OOP** comparisons, since those logics are abstracted to easily reusable functions on Functional Programming.

A good example of this could be again the collections API functions mentioned before. I.e: mapping a collection (or calling map() over it) means applying a mapping function to each one of its elements and returning a new collection containing all those. So there are 2 separated parts here:

* Abstraction of mapping generic behavior, which is appliable over any collection. That would be the `map` high order function by itself.
* Implementation details for the current execution are delegated to the function being passed as an argument.

So if you just look a bit behind the scenes, you will notice that Kotlin is implementing by itself some approaches very related to FP langs.

**Does it mean it’s an Functional Programming language?** Probably not, or probably yes ¯\_(ツ)\_/¯. I mean, what a functional programing lang is, is not clearly defined and it depends on where you look at or which posts do you read. The boundaries for that could be a bit blurry. But honestly, that is not such important here.


In the end, truth is Kotlin is kind of “enabling” this higher abstraction level paradigm just by adding some interesting built-in features. So if we wanted to achieve it, we could.

### But should we?

It’s a well funded question, since we could just keep doing OOP and that’s all, given that it has been working for us since ever. So why to change?

Well, let’s expose three really important benefits from Functional Programming to start with.

### Referential Transparency

A function should be completely clear and straightforward about what’s it asking for, and what’s it giving in return. That means **side effects are not welcome** here, since those break this term. Just by looking at a function return type and it’s parameter types, I must be able to know exactly what can I get out of it. So the goal is to make the function completely explicit, including all the possible return cases.

**When you are writing public apis you should always expose types** (by convention). That means, even if the lang is allowing you to avoid them on return types of functions thanks to type inference, for example, you should still add them. Type inference is great, but public APIs from functions should not be poisoned with it, since that breaks public API contract readability, so therefore referential transparency. At the same time, please, feel free (and I promote it) to use type inference in any other cases, like function bodies.

### Purity

If I call a function one billion times, and I pass it the same parameter values every single time, I should get the exact same result every single time. That means the function is not performing any side effects. Its not doing anything unexpected behind the scenes. It is not trying to access or modify any external state.

That is called a **pure function**, and those are ideal to do functional programing, since its mainly based on pushing side effects out of the equation. We don’t want side effects to introduce non-determinism into our function tree. So we will find ways to get rid of state and side effects on all those layers of our system that do not really require having those.

So a code based on purity is a completely predictable code, and hidden bugs cannot happen so easily. Think about this: Usually in OOP, bugs are related to state change or race conditions provoked by it. Some state is being changed somewhere behind the scenes when it shouldn’t, and that‘s the root of the problem.

**Purity means determinism, which means higher and easier testability**.

### Concern Separation

Looking at FP programs, there is a concept which is present every single time. FP programs / systems are divided in two different parts:

* Compose a tree of declarative and deferred computations to implement your system logic. It’s not gonna be run yet, since we are just declaring all those nested and ordered computations that will take place the moment we actually decide to run it. On this step, you are completely open to depend just on abstractions (like *Typeclass* defined behaviors or even *Free* lifted operations. You will have details about both techniques in future posts).
* Runtime evaluation: After providing the whole execution tree we can decide to run it. At that moment you usually provide all the semantics / implementation details needed for it. So at that very moment and not before, you are resolving the ambiguity imposed by the abstractions.

This two steps are present on **any** FP style. And they are a quite big improvement on how we model our programs, since we are now open to (and able) to define them in a completely declarative and abstract style. So we could validate and test our whole execution tree almost in an end to end black box scenario just by running the composed computation passing in the implementation details we wanna choose on the evaluation phase.

Those could contain some mocks for dependencies causing side effects :). We want to achieve isolation from frameworks, after all.

### Does Kotlin have everything needed to go FP?

No, it doesn’t. The language still lacks some key features that we would require in order to go for type safe pure Functional Programming.

Some of those, and really important ones, could be **Higher Kind Types**, **Typeclasses**, and a lot of functional constructs and abstractions. Functional programming in a complete way cannot be applied without all those.

[There is a KEEP](https://github.com/Kotlin/KEEP/pull/87) open by [Raúl Raja](https://www.twitter.com/raulraja) for the Kotlin team to ask for including Typeclasses and HKs on the language. There is a really interesting dicussion inside, and the official Kotlin team from JetBrains is evaluating this option.

This doesn’t necessarily mean it will be implemented, but it is being taken into good account by them. So please, vote there if you would like to have those features on Kotlin!

So, since there is a lot of work to do, and since we found out good approaches to implement / emulate those things, we decided to create a library for it: Arrow ([arrow-kt.io](https://arrow-kt.io/))

> Arrow brings Functional Programming types and abstractions to the Kotlin language.

In the following posts, I will be describing four different FP styles applied over an Android app, including all their benefits and details step by step. It will be valid as documentation about **how to use Arrow on top of Android** to go Functional Programming.

In the meantime, you can follow me on Twitter [@JorgeCastilloPr](https://www.twitter.com/JorgeCastilloPr), where I speak a lot about this topic and many other ones. I will announce new posts there! 🎉
