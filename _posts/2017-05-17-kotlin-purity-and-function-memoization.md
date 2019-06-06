---
layout: post
current: post
cover: assets/images/neurons.jpeg
navigation: True
title: Kotlin purity and function memoization
date: 2017-05-17 07:00:00
tags:
class: post-template
subclass: 'post'
author: jorge
---

Let’s learn about the benefits of "purity" and "pure functions", and how it affects caching.

A pure function is a function that **just operates over it’s input arguments to provide a result**. It has no side effects, which means that the function itself is not provoking any external effects that you cannot control or **you don’t expect**. It’s not modifying any external state behind the scenes. So if you ran the function one trillion times for the same input arguments you would get the same result one trillion times.

A pretty common example of pure function usage uses to be the one about JDK Math class functions.

```kotlin
val fifteen = Math.abs(-15)
val arcTangent = Math.atan(90.0)
val arcSine = Math.asin(45.0)
val eight = Math.sqrt(64.0)
```

All those functions would always return the same results for those given arguments, with no exceptions.

In java it’s easy to create pure functions. You just mark those functions as `static` so those cannot access any state since they aren’t bound to any enclosing class instance. The compiler itself is going to forbid modification or access to any **non static** external variables from outside the function, but **you shouldn’t either modify global static ones**.

[Referential Transparency](https://wiki.haskell.org/Referential_transparency) is a concept definitely present on pure functions. I learned this concept from a [very interesting talk about Functional Error Handling by Raul Raja](https://www.47deg.com/presentations/2017/02/18/Functional-error-handling/).

Referential Transparency means that the function does exactly what it promises when you call it, no more, no less. This concept is pretty beneficial for testability. If your code is most likely based on pure functions, that means it’s also more predictable. And **predictable code is normally easier to test**. Side effects break referential transparency.

---

![Hermione](assets/images/hermione.gif)

*Hermione found that purity doesn’t fit pretty well inside her last released app. She might use some spells to fix it.*

If you are an Android developer you know about the need to apply side effects at some point, otherwise you couldn’t render things on screen or query to an API. Well, you can **apply those effects to the edges imposed by your system**. Talking about Android, we would be talking about lifecycle callbacks where Android IoC gives us the control. We will talk a bit about this again in the following paragraphs.

The rest of your architecture could be totally based on **purity-based transformations over the data**, if you wanted.

If you pass all your dependencies as parameters all the way, you will find that you are progressively **getting rid of state** across your architecture except for the edge imposed by it, that would be where the inversion of control is applied (lifecycle callbacks).

In Java, we normally have state all over the place just because OOP, and because we also decide to inject dependencies through constructors / fields and keep their references retained inside class instances. But that’s our choice on that. It doesn’t mean we have a requirement to have state, neither an instance. The only meaning of this situation is that the `DI` framework we are using is kind of forcing us to have state.

If you start moving towards this mindset, you will find that you could remove a lot of classes and focus more on functions as **first class citizens**.

Also, **having no state means no need to switch those pieces at testing time**. You will be completely safe by using your real production code for all those chained functions inside your tests, since there aren’t any problematic side effects creating the need for test doubles. And the testing environment should always replicate production code as much as possible, otherwise you wouldn’t be writing the right tests, isn’t it?

This enables you to test your app almost end to end in a **black box scenario** just by switching your View implementation (Android) and a couple more side effecting dependencies by test doubles. Purity predictability will make you capable of exercising a bunch of chained functions and easily know what to expect in return for your assertions.

Also, your production dependency trees would be much shorter. Your switchable dependencies would be just the required ones to apply side effects, but other than that your connection between the inner layers would just be a bunch of pure functions calling each other to apply transformations over the data. Something that does not requires to be passed as an injected dependency.

But we also have to look at the possible **disadvantages and caveats of purity**.

### 😭 Performance consequences

By getting rid of the state we are also sacrificing something important. Let’s say that we have the need for a memory cache of items loaded from the API.

First of all, we see a problem on querying the API, since that’s a side effect.

Well, a good approach to avoid breaking purity would be to wrap the side effect inside an `IO` Monad to defer its execution. This means that we are going to compose a chain of transformations over the data plus the deferred data retrieval by the `IO` Monad, and we will run the whole chain from the edge imposed by the system, right when we are ready for it. That will happen on our `onCreate` or `onResume` Activity / Fragment method, most likely.

So all the side effects are applied on Android lifecycle callbacks, inside the view implementation. So we are safe on this.

But what happens when we have to do some expensive calculations in some of the layers composed just with pure functions? Also, what would happen if we are server side and the mentioned function to do the calculation is called thousands of times?

Let’s have a look on the following code snippet. It’s a class instance with some state. It’s capable of calculating the factors of any possible integer:

``` kotlin
class FactorCalculator {
    val sumCache = hashMapOf<Int, Int>()
    val factorCache = hashMapOf<Int, List<Int>>()

    fun sumOfFactors(number: Int) = sumCache.getOrPut(number, {
        factorsOf(number).sum()
    })

    fun isFactor(number: Int, potential: Int) = number % potential == 0

    fun factorsOf(number: Int) = factorCache.getOrPut(number, {
        (1 to number).toList().filter { isFactor(number, it) }
    })
}
```

The `sumOfFactors` function returns a sum of all the factors from any `Int`. That value is calculated by the `factorsOf` function, which takes a range from `1` to the current `number`, converts it to a list and then filters to get just the values that are actually factors of the given `number`.

Both functions rely on a `HashMap` structure to cache already calculated values results for previous inputs, just to save some CPU cycles and memory. Using the `map.getOrPut()` function from Kotlin we simplify the code a bit.

But we have state here, so we don’t have purity. Our functions are applying side effects over the `HashMap` structures. So, let’s try to convert this class to a bunch of pure functions that just operate over their input parameters:

``` kotlin
fun sumOfFactors(number: Int) = factorsOf(number).sum()

fun factorsOf(number: Int) = (1 to number).toList().filter { isFactor(number, it) }

fun isFactor(number: Int, potential: Int) = number % potential == 0
```

We dropped the concept of class here, as it’s no longer needed. But during the process, we didn’t have any other choice than to drop the state since we want those functions to be pure. So we lost our memory caches forever!

![Sad Afflec](assets/images/sad_afflec.gif)

*Affleck realized at the bar what he lost on that refactor pushed to production hours ago*

Well, Affleck has **good reasons to be sad**, since his refactor was part of some very often used code from server side, and server response timings are going to be damaged so badly because of it.

Looking back at the previous snippet, it seems like we don’t have any simple possibility to keep a memory structure storing those already calculated values without provoking side effects.

Well, that’s not really true and here is were the need for a feature like **function memoization** comes in 🎉.

### 🤔 Function memoization

It’s a feature commonly built into some programing languages like Groovy, which **enables automatic caching of recurring function-return values**.

The idea around the concept is to provide a memory cache at a function level that stores the already calculated function result values. So if we get this feature we wouldn’t need a class instance and a field reference to store it for us anymore, it would be held as part of the function language meta-level.

Languages that do not implement this out of the box don’t have hard times implementing it, although they can lose some syntactic sugar during the process.

Memoizing a function is a **metafunction** application. That means we are doing something to the function itself and how it works at a language meta level and not to the function results or it’s code implementation.

[Here you have a pretty clear article](https://objectpartners.com/2014/01/28/memoization-in-groovy/) by Brendon Anderson about how function memoization works in `Groovy`. The code examples inside of it are pretty straightforward. I will paste one here to reduce noise so we can talk about it:

```groovy
def myClosure = { Integer x ->
    println "My Closure argument $x"
}.memoize()

myClosure 3
myClosure 4
myClosure 3
myClosure 4
```

You define a closure and then call the language built-in `memoize()` method on top of it to get a memoized version for the same function. This is the resulting output after running the function those 4 times:

```groovy
My Closure argument 3
My Closure argument 4
```

So, quoting the article: *“You can see that the code inside the closure was executed only once for each distinct input parameter.”*

That’s because the memoized version of the function is returning already cached results and not recalculating them if it’s not needed. It just replaces the resulting value with the cached one if the function was called with the same parameters than a previous call so the function code is not really run.

**This can be done because the function is pure**, and we can tell that we will always get the same result for the same input parameter. Otherwise we couldn’t feel safe about replacing it’s returning value with it’s cached version.

So, how could I achieve the same behavior in `Kotlin`?

### 🎁 Kotlin implementation

Sad story here: Memoization isn’t a built-in feature for `Kotlin`. I will not paste Ben Affleck here again, but I can pretty much imagine your face.

Anyways, we can still implement it by ourselves so…

![why so serious](assets/images/why_so_serious.gif)

Take a look at this simple Kotlin implementation of the feature:

```kotlin
class Memoize1<in T, out R>(val f: (T) -> R) : (T) -> R {
    private val values = mutableMapOf<T, R>()
    override fun invoke(x: T): R {
        return values.getOrPut(x, { f(x) })
    }
}

fun <T, R> ((T) -> R).memoize(): (T) -> R = Memoize1(this)

val memoizedSumFactors = { x: Int -> sumOfFactors(x) }.memoize()
```

First, let’s look at the class on top. I have used a class here so it becomes clearer to read, but I could also create an anonymous object extending the correct function type `(T) -> R`.

On this basic example I am just covering the case of functions with a standalone parameter that also return a value. But the same strategy could be applied for any number of function arguments.

Taking a cautious look, you will notice that the `Memoize1` class receives as function with the type `(T) -> R` as a property. Then I overrided the `invoke` method to be able to make it’s implementation rely on a `MutableMap`. The map is used to hold the already calculated values as an in memory cache, so the function will return a previously calculated result if the input value is contained in the map. Otherwise, the function will be run and it’s result will be stored in the map right after that.

> Overriding invoke is like overriding the main application of the function, which is it’s main constructor. In Kotlin, a call with the form a() is converted to a.invoke(). You can also invoke any function by using a.invoke() by yourself. This is equivalent to apply() method from Scala.

I also added an extension method for all the functions with the form `(T) -> R`, for any `T` and `R` types. Thanks to that one, you can call `memoize()` over any function that receives a value and returns a result. You will get it’s memoized version in return.

```groovy
val memoizedSumFactors = { x: Int -> sumOfFactors(x) }.memoize()
memoizedSumFactors(3) // this would be the only one calculated
memoizedSumFactors(3)
memoizedSumFactors(3)
memoizedSumFactors(3)
memoizedSumFactors(3)
```

As you can see on the snippet, by using this code you would get your function body run just once, and for the resting calls for the same input value we would get the cached result instead.

So here we got what we where looking for! 🎊

---

After writing this article, found out that the really well known library [funKTionale](https://github.com/MarioAriasC/funKTionale) from [Mario Arias](https://github.com/MarioAriasC) already [implements memoization](https://github.com/MarioAriasC/funKTionale/blob/8c36f2d411691f0fe1da62e720d0a0df2834da2d/funktionale-memoization/src/main/kotlin/org/funktionale/memoization/namespace.kt#L28) for Kotlin. It’s gonna become a really good example of how to enable memoization for functions up to 22 parameters.

As you can see, [he uses objects extending the required function types](https://github.com/MarioAriasC/funKTionale/blob/8c36f2d411691f0fe1da62e720d0a0df2834da2d/funktionale-memoization/src/main/kotlin/org/funktionale/memoization/namespace.kt#L29), instead of named classes, to provide a simpler implementation.

You might think that the code is pretty boilerplate with all those extension functions and classes for the handler. This sort of *“predef”* files to define language extensions are pretty typical on functional programing. Truth is that thanks to this simple file that you could copy to any of your projects, you enable memoization for any possible function you want to use.

So it should be worth it, isn’t it?

If you want to avoid the need to implement memoization by yourself, and now that you already know what’s the concept about, you could add the dependency of *funKTionale* to your project. If you are moving your app towards a more functional architecture, this library will be handful for you, and it’s worth it. Thanks to Mario for putting his best efforts on it.

---

This should wrap things up for now. If you are interested on `Kotlin` and possible functional approaches around the language, you could want to take a look to the following articles I wrote recently about tail recursion and *Dependency Injection* using the `Reader` monad. You might agree or disagree, but those are interesting concepts pretty handful to understand:

* [Tail recursion and how to use it in Kotlin](https://medium.com/@JorgeCastilloPr/tail-recursion-and-how-to-use-it-in-kotlin-97353993e17f)
* [Kotlin Dependency Injection with the Reader Monad](https://medium.com/@JorgeCastilloPr/kotlin-dependency-injection-with-the-reader-monad-7d52f94a482e)

---

Please, feel free to add me on Twitter [@jorgecastillopr](https://twitter.com/jorgecastillopr) to discuss anything related (or not even related!) to this article. I usually post and retweet about `Kotlin` and any other Android development and functional related posts.
