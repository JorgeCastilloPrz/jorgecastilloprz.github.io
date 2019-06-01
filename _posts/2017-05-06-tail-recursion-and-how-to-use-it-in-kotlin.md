---
layout: post
current: post
cover: assets/images/tail_recursion.jpeg
navigation: True
title: Tail recursion and how to use it in Kotlin
date: 2017-05-06 12:36:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

Learn the concept of tail recursion and how to get the most out of it in the Kotlin language.

The concept of tail recursion is not linked to Kotlin or any concrete language but to all of them. It’s a generic concept, like standard recursion. Let’s just start reading a pair of basic examples of imperative and recursive iterations over collections to get in rapid context. We will use the pretty standard filtering method present in almost all the modern and not so modern languages:

### The imperative way

This is how a `filter` function could look like if we think about it in an imperative way:

``` kotlin
fun <T> filter(l: List<T>, f: (T) -> Boolean): MutableList<T> {
    val res = mutableListOf<T>()
    l.forEach { if (f(it)) { res += it  } }
    return res
}
```

I am pretty sure you are now thinking: “this guy is astoundingly stupid”, and for god sake that’s correct. But let me just hide to you the fact that we could have done this: `l.filter(f)`, since the language already provides that function out of the box 🤐.

Looking at the code snippet, we can see how we prepare an empty list and just append each element from the source list to the final one if it satisfaces the `f` predicate.

With this approach, we are **managing the state inside the function** by ourselves, instead of ceding that responsibility to the runtime. We need to create a list and maintain it until the end of the loop to be able to return it.

But, do we really need to do it? As you know, state can be a source of problems.

Since we have functional colors in all the modern languages, we should benefit from that and try to move to more functional oriented approaches using pure functions. That also means to try to get rid of the state, side effects, and pass more things as parameters 🙌.

All the languages have their ways to cede operations to the runtime in order to avoid non required complexity to developers. That’s how garbage collectors were born. And that’s how programing evolves. Of course, we can do something to improve our filtering function thinking that way. Let’s start by **using recursion** to make it evolve. We will keep the state alive for now.

![pikachu](assets/images/pikachu.gif)

That's probably the weirdest pokemon evolution I found on the Internet.

Excuse me, here you have the nerdest pokemon evolution I found on the Internet.

### Going the recursive way

This is how the `filter` function could look like thinking in a standard recursive way:

``` kotlin
fun <T> List<T>.tail() = drop(1)
fun <T> List<T>.head() = first()

fun <T> filter(l: List<T>, f: (T) -> Boolean): List<T> {
    if (l.isEmpty()) {
        return listOf()
    } else {
        return if (f(l.head())) {
            listOf(l.head()) + filter(l.tail(), f)
        } else {
            filter(l.tail(), f)
        }
    }
}
```

> To improve readability I have created two extension methods to provide `tail()` and `head()` methods to any `List`, since those are not available out of the box with those names. Function call `drop(1)` implements the same behavior as `tail()`, since it returns a new list with all the elements except the first one or an empty list it was already empty. And for `head()` function I just used `first()`.

This code snippet could be done in a more expressive way most likely. But in purpose of having simple examples, let’s leave it as it is.

As you can see in the snippet, we are leaving the list state to the runtime using recursion. We don’t need to create and maintain the reference to it anymore, that’s going to be handled in memory by the program runtime. We just limit ourselves to append an element on each recursion, again just in case it satisfies the `f` predicate.

Nothing new, huh ?

But **recursion can also bring big problems to the table**. If we were working over quite big lists here, we could get a `StackOverFlowException`. By history, the call stack has been the main problem of the standard recursion for all the languages. So it’s an interesting feature, but you need to know when and also how to use it. You should have that always in mind.

And we are talking about the how in a second, but first let’s take a look about why we need to use the call stack for this type of recursion.

### The need of a call stack

When we look at the last code snippet, we find the need for the language of building a call stack to be able to rewind it when we reach the last recursion. As you know, the program just goes deep there and tries to resolve all the nested function calls until the very last one. Then the stack is rewinded.

The real need for the stack here is because **there are still operations to be done after the last recursion**. In this case, the runtime replaces each recursion call in reverse order, from the deepest to the outer ones, by it’s resulting list of elements. When it reaches the outer most level, then it can proceed to actually call the method to build the list.

But there are ways of avoiding this. For the `filter` function example, the idea would be to try to build the list on each recursion and pass the temporary list to the next step, so the previous call in the stack becomes useless and could just be discarded. If we can achieve this for all the steps, we will reach the final recursion with the complete list already built, so **there would be no need to rewind the stack anymore, therefore we wouldn’t actually need to maintain the stack**. We could just return that value to the top most call level.

That’s also called **tail recursion**, and we are talking about that in detail now.

![another pokemon](assets/images/weird_pokemon.gif)

### The tail recursive way

Before explaining how we can tell the compiler to optimize a function that uses tail recursion, let’s take a look at the following example:

``` kotlin
fun <T> filter(l: List<T>, res: List<T>, f: (T) -> Boolean): List<T> {
    if (l.isEmpty()) {
        return res
    } else {
        return filter(l.tail(), if (f(l.head())) { res + listOf(l.head())} else { res }, f)
    }
}
```

That line almost didn’t fit on the snippet 😵. But this is Kotlin, so one liners everywhere moth** f**** 💪🏿
If you look carefully, you will realize that this function is calling itself again as it’s last statement. Also, we appending to the resulting list instead of prepending to it. For each recursion, we just call the filter method again passing in the tail of the current source list, and a temporary built list with the current item appended to it just when necessary.

As a result, we will be building our resulting list meanwhile we do all the recursions.

That means that now **the end case can return the already complete resulting list**, so there are no more operations to be done during or after the rewind phase. We built the stack just because that’s the way the JVM works, but the already executed calls become useless right after we run the next step in the chain. So the end case return value gets passed as it is during the whole rewind phase up to the outer most level.

This approach can be used to **let the compiler optimize our recursive methods** for some languages, just in case they support it. Kotlin is one of those. Basically, the compiler optimization for tail recursive functions removes the possibility to overflow your stack forever. So what you get is a **stack safe recursion**.

To use this compiler optimization you just need to tag your tail recursive function with the **tailrec** reserved word.

``` kotlin
tailrec fun <T> filter(l: List<T>, res: List<T>, f: (T) -> Boolean): List<T> {
    if (l.isEmpty()) {
        return res
    } else {
        return filter(l.tail(), if (f(l.head())) { res + listOf(l.head())} else { res }, f)
    }
}
```

By doing this, the compiler becomes capable of translating this to a simple loop during compilation, and that’s always stack safe since everything will occur under the same call inside the stack.

A good statement extracted from the Kotlin documentation that defines the requirements for a recursive function to be able to be declared as **tailrec** is the following:

> To be eligible for the **tailrec** modifier, a function must call itself as the last operation it performs. You cannot use tail recursion when there is more code after the recursive call

Another important thing to notice, is that we are just passing the resulting list as a parameter for all the recursions, so the state is not maintained by us or the runtime anymore, but just removed forever. We just operate over input parameters to provide results. That’s pure functions. There is no state and no side effects here.

![no power here](assets/images/nopowerhere.jpeg)

[Here you have all the details about tail recursive functions from the official docs](https://kotlinlang.org/docs/reference/functions.html#tail-recursive-functions).

---

### Extra bullet

Tail recursion compiler optimization is not supported by some languages. Some of those use a technique called “trampolining” to emulate this behavior. This technique is based on returning a trampoline in your function, which is just an structure capable of running the same function again until it returns anything that is not a trampoline. That would emulate the end case of a recursion. At that point, the resulting value is taken as the overall resulting value for the whole chained function execution. This is also not stacking calls because when the current call is done, it dies to give birth to the next one thanks to the trampoline.

[Here you have a really interesting article](https://espinhogr.github.io/scala/2015/07/12/trampolines-in-scala.html) about how you can use trampolines in Scala to achieve tail recursion for scenarios where you cannot use a normal tailrec (like recursions with 2 or more functions).

Please, feel free to add me on Twitter [@jorgecastillopr](https://www.twitter.com/JorgeCastilloPr) to discuss anything related (or not even related!) to this article. I usually post and retweet about Kotlin and any other Android development and functional related posts.

Other than that, just buy a pool trampoline. But it will not be stack safe!

![simpsons](assets/images/trampoline.gif)

oh my.
