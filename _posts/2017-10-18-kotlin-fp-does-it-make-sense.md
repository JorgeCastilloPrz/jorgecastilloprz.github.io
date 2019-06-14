---
layout: post
current: post
cover: assets/images/flowerfield.jpeg
navigation: True
title: Kotlin Functional Programming, Does it make sense?
date: 2017-10-18 15:18:00
tags: [kotlin, fp]
class: post-template
subclass: 'post'
author: jorge
---

Functional programming can feel like a big wave of fresh air over a beautiful landscape at sunset. The flowers are literally us. 😍 🌼 (maybe?)

Ok no jokes: it’s clearly worth it!

### Is Kotlin a FP lang?

When we think about Kotlin, OOP is always implicit. But that’s not strictly necessary. Kotlin is still **open to a different paradigm** thanks to some of its built-in features. I’m referring to FP here.

If you take a careful look at the built-in features, you will find some very obvious ones like *higher order functions or the power to use functions as first class citizens*. That means you're able to treat functions as values in Kotlin. This feature is very related to **function oriented styles**, even if that does not strictly mean Functional Programming. But it opens a whole new world of approaches for you.

Having the possibility to pass functions as arguments, return them as other functions results, or store them in variables or lists for later usage, opens the language to new techniques like **deferring execution**. If you return functions on your different layers instead of already computed values, what you get back from it is a composed deferred computation, or in other words, **lazy evaluation**. Your computations are deferred in time, still not run, so you keep control over when to do that.

This is completely related to FP, and you will see why in the following lines.

Also *functions as first class citizens*, as you already know, means you can just use functions the same way and on the same places you were using classes in Java. That means you can declare those at a package level, if you want to. Because honestly, do we always need to play with instances all the time?, or is it kind of “imposed” by our paradigm and the libraries we use?

Actually if you take a careful look at the language you will find tons of additional features that you can find on many functional languages: Type inference, operator overloading, good support for syntax extensions (implemented by *extension functions*), collection combinators as `map`, `flatMap`, `fold`, `reduce`…, or even **algebraic data types** (sealed classes + data classes).

The language is even abstracting some logics and behaviors to functions, and not to classes, as you would do in OOP. That’s something very usual on **FP vs OOP** comparisons, since those logics are abstracted to easily reusable functions on Functional Programming.

A good example of this would be again the collections API functions mentioned before. I.e: mapping a collection (or calling `map(f: (A) -> B)` over it) means applying a mapping function to each one of its elements and returning a new collection containing all those. So if we split this behavior in two parts:

* Abstraction of mapping generic behavior, which is appliable over any collection. That would be the `map` high order function by itself, so it applies a given `f` lambda for all the elements in a collection.
* Implementation details for the current execution are delegated to the function being passed as an argument (`f`).

So recaping a bit, looks like Kotlin implements some approaches very related to FP langs.

**Does it mean it is or can be considered a Functional Programming language?** Probably not, or probably yes ¯\\_(ツ)\_/¯. I mean, what a functional programing lang is, is not clearly defined and it depends on where you look at or which posts you read. The boundaries for that could be a bit blurry. But honestly, I think that is not such important here.


At the end of the day, truth is Kotlin is “unlocking” the chance to do pure Functional Programming just by adding some interesting built-in features. So if we wanted to achieve it, we definitely could.

### But should we?

It’s a well funded question, since we could just keep doing OOP and that’s all, given that it worked for us since ever. So why to switch?

Well, let’s expose three really important benefits from Functional Programming to start with.

### Referential Transparency

A function should be completely clear and straightforward about what’s it asking for, and what’s it giving in return. That means **side effects are not welcome** here, since those break this term. Just by looking at a function return type and it’s parameter types, I must be able to know exactly what can I get out of it. So the goal is to make the function completely explicit, including all the possible return cases.

**When you are writing public apis I'd suggest to always expose types** (by convention). Even if the lang allows you to avoid them on return types thanks to type inference. Type inference is great, but public APIs from functions should not be affected by it, since that **breaks public API contract readability**, hence referential transparency. At the same time, please, feel free (and I promote it) to use type inference in any other cases, like function bodies.

### Purity

If I call a function one billion times, and I pass it the **same argument values** every single time, I should get the exact **same result** every single time. That means the function is not performing any side effects on its body. Its not doing anything unexpected under the hood. It is not trying to access or modify any external state, neither using any third party APIs that could potentially throw.

For side effects we understand things that escape your program's logic control, like printing to console, rendering to a display, sending metrics to an analytics system in a server, performing an HTTP request, storing something on a cache (no matter whether it's persistence or memory), and much more. As soon as your functions provoke changes in the observable world or try to read from it, they contain side effects.

Effects can be slow, fail (throw), be cancelled, mutate the state of your program... etc. They leverage ambigüity within your program.

Functional Programming is all about purity, since its mainly based on pushing side effects out of your architecture. We don’t want side effects to introduce non-determinism into our call stack. Because that removes the ability to do "local reasoning" over our code. If we can't be certain of what a method is gonna do for all the cases (no exceptions, pun intended!) then we can't really reason about it in a deterministic way.

One good example of this could be compiler optimizations. If every time your code calls `add(1, 2)` it returns `3`, compiler can infer that and replace all calls like `add(1, 2)` in your code base by `3`, the actual result. Compiler optimizations are just possible when code is deterministic, given they work by inferring behaviors over code.

So overall we will always try to find ways to get rid of state and side effects on all your architecture layers where you do not really require having those.

To achieve that we can do something we've been doing since ages ago in OP: **Depend on abstractions**. When your architecture targets abstractions and you leave implementation details for runtime (**side effects are implementation details, keep that in mind please!**), you achieve a complete pure architecture. That's how you achieve what's called **Concern separation**.

So a code based on purity is a predictable code, and hidden bugs cannot happen easily. Think about this: Usually in OOP, bugs are related to state change or race conditions provoked by state change on arbitrary (unexpected) points of your system. Some state is being changed somewhere behind the scenes where it shouldn’t, and boom that‘s the root of the problem.

> Purity means determinism, which means higher and easier testability.

### Concern Separation

Looking at FP programs, there is a concept which is present every single time. FP programs / systems are divided in two different parts:

* **Algebras:** Compose a tree of declarative and deferred computations to implement your system logic. That's usually done using **algebraic data types** that define the operations your program is able to perform. This stack of computations **is not gonna be run yet**. It's actually just declared and we can already reason about it, but it's waiting for us to decide when we're ready to run it. On this step you'd usually just target abstractions (like *Typeclass* defined behaviors or even *Free* lifted operations. You will have details about both techniques in future posts). So your code still does not know anything about concrete semantics, just completely abstract behaviors.
* **Runtime:** After providing the whole execution tree we can decide to run it. At that moment you usually provide all the semantics / implementation details (side effects) needed for it. So at that very moment and not before, you are resolving the ambiguity imposed by the abstractions.

This two steps are present on **any** FP program. And they are a quite big improvement on how we model our programs, since we are now open to (and able) to define them in a completely declarative and abstract style. So we could validate and test our whole execution tree in an end to end black box scenario just by running the algebras (our program) by passing in the implementation details that are more convenient for us at **runtime**.

Those implementation details will be the side effects, and **will be the pieces we will end up replacing in tests by test doubles**. We want to achieve isolation from frameworks, after all. For anything else we can still use our production code, since it's gonna be all **pure functions that don't need to be mocked**. We can safely use the production ones as they are, being able to exercise a big chunk of our production codebase and just mock the edges and a couple more pieces. That indeed unlocks black box testing.

### Does Kotlin have everything needed to do pure FP?

No, it doesn’t. The language still lacks some key features that we would require in order to go for type safe pure Functional Programming.

Some of those, and really important ones, could be **Higher Kinded Types**, **Typeclasses**, and a lot of functional constructs and abstractions. Functional programming in a complete way cannot be applied without those.

[There is a KEEP](https://github.com/Kotlin/KEEP/pull/87) open by [Raúl Raja](https://www.twitter.com/raulraja) for the Kotlin team to ask for including Typeclasses into the language. There is a really interesting dicussion inside, and the official Kotlin team from JetBrains is evaluating this option.

This doesn’t necessarily mean it will be implemented, but it is being taken into good account by them. So please, vote there if you would like to have those features built in in the Kotlin compiler!

So, since there is a lot of work to do, and since we found out good approaches to implement / emulate those things, we decided to create a library for it: Arrow ([arrow-kt.io](https://arrow-kt.io/))

> Arrow brings Functional Programming types and abstractions to the Kotlin language.

Stay tunned for more posts about Functional Programming in Kotlin. There's much more to come!

In the meantime, you can follow me on Twitter [@JorgeCastilloPr](https://www.twitter.com/JorgeCastilloPr), where I speak a lot about this topic and many other ones. I will announce new posts there! 🎉
