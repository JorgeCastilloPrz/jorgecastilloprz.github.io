---
layout: post
current: post
cover: assets/images/piano.png
navigation: True
title: Support for Kotlin SAM in release 1.4
date: 2020-08-15 11:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

About Kotlin SAM support and how it's also possible for Kotlin interfaces starting from version 1.4.

### 🤷‍ SAM?

This feature also exists in other JVM languages like Java 8 or Scala. SAM stands for Single Abstract Method and it refers to interfaces with a single **non-default** method. Those that are frequently used as **callbacks**.

These interfaces are also called "functional interfaces" frequently because their only purpose is to be used as a function. That is actually the name they have in Java.

### 🍬‍ Java sugar

Kotlin can sugarize any Java interfaces that follow that pattern into lambdas to leverage interoperability. This process is called **SAM Conversion**.

The main goal for this feature is interoperability. The ultimate idea is that you can call a Java SAM as if you were calling a simple function.

SAM conversions work in both directions:

* Java SAM 👉 Kotlin lambda.
* Kotlin lambda 👉 Java SAM.

They are possible when the input and output types for the Java SAM and the Kotlin function match. Let's use a simple example. Here's a Java SAM interface.

```java
public interface Consumer<A> {
  void consume(A a);
}
```

If we tried to use this `Consumer` from Kotlin as if it was Java, we'd probably go for an anonymous object:

```kotlin
val logger = object : Consumer<String> {
  override fun consume(a: String) {
    println(a)
  }
}
```

When you do that, the Kotlin idea plugin will warn you about the chance to convert it to a lambda:

<img width="820" src="/assets/images/kotlinsam1.png"/>

As soon as you do this, the code will turn into the following:

```kotlin
val logger = Consumer<String> { a -> println(a) }
```

Much better, isn't it? There is only a little con. When used to create an instance of the SAM we are forced to provide the interface name explicitly. You can't avoid that, even if you provide an explicit type on the left side.

That is different when you use it in an input position, like a function parameter. Here, you can still be explicit or simply omit the interface name. These two examples are equivalent:

```kotlin
// Explicit interface name
program.consume(Consumer<String> { a -> println(a) })

// Omit interface name
program.consume { a: String -> println(a) }
```

If your Java class has multiple method overloads that take different SAM interfaces, you can always provide the type explicitly for disambiguation.

SAM conversions were **only possible when interoperating with Java functions** before Kotlin 1.4. This was a language design decision, since this feature existed only for interoperability with Java, given Kotlin already provides support for function types and function literals.

### 🍭 Kotlin sugar

I will later explain why that decision changed, but starting on [the new stable release 1.4](https://github.com/JetBrains/kotlin/releases/tag/v1.4.0), SAM interfaces are also supported for Kotlin. Now you can write your interface in Kotlin and SAM conversions will also work.

Let's say we had this Kotlin interface:

```kotlin
interface Producer<A> {
  fun produce(): A
}
```

Before Kotlin 1.4 this was the only way to create an anonymous instance for it:

```kotlin
val heyProducer = object : Producer<String> {
  override fun produce(): String = "Hey"
}
```

And the same was required for passing an instance to a Kotlin method:

```kotlin
fun <A> produceNew(producer: Producer<A>) = producer.produce()

produceNew(object : Producer<Int> {
  override fun produce(): Int = (0 until 10).random()
})
```

Starting on Kotlin 1.4, SAM conversions also work for those scenarios. First thing you'll need to do is flag your functional interfaces as `fun interface`. Kotlin made this an opt-in and explicit choice so not all interfaces with a single abstract method work as functional interfaces by default.

```kotlin
// Kotlin 1.4+
fun interface Producer<A> {
  fun produce(): A
}
```

And now you can use it like this:

```kotlin
val heyProducer = Producer<String> { "Hey" }

produceNew(Producer<Int> { (0 until 10).random() })
```

Some interfaces widely used from the Kotlin stdlib are also flagged as `fun interface` now. Some of those are `Comparator`, `ReadOnlyProperty` and `PropertyDelegateProvider`.


### 🤔 Why would I use it for Kotlin interfaces?

Kotlin has support for function types and function literals (lambdas), so that is a reasonable doubt to have.

That is precisely the same doubt the Kotlin team had for not introducing those before. They just realized now there are some scenarios where it could be convenient. [Here you have the community discussion](https://youtrack.jetbrains.com/issue/KT-7770?_ga=2.103945408.1399009746.1597478338-1610252576.1591121905&_gac=1.256115449.1597482077.CjwKCAjwj975BRBUEiwA4whRByZeeWUfKevdonRWUAmo11kB3Q4ruIf4kMjpjkqWjrfJgr2GWA6nghoC47kQAvD_BwE) that triggered the feature development later on.

### Memory implications

Lambdas in Kotlin can reference external **non-final** variables, not like in Java. When that happens, Kotlin wraps those in `Ref` and captures them for their usage within the lambda. More details on that [on this post by Tompee Balauag](https://medium.com/tompee/idiomatic-kotlin-local-functions-4421f86ac864).

When Kotlin lambdas are not capturing (they only access final variables), they can be instantiated **as a singleton object per application**, so same instance can be reused every time without an additional cost on memory allocation.

For interfaces in the other hand that's different, since Kotlin creates a new object for each call. There are interesting gotchas regarding this [like the one described on this post by Vasya Drobushkov](https://medium.com/@krossovochkin/kotlin-java-interop-function-references-and-sam-conversions-3d0cd36f7967), so it's good to keep this in mind.

### 💡 Extra bullets

An interface with a single `suspend` function cannot be used as a SAM. Those currently have some issues during the code generation phase at this point, so for now [they are prohibited](https://youtrack.jetbrains.com/issue/KT-40978).

SAM conversions **only work for interfaces** as of today. Not for abstract classes.

* [Official - SAM conversions for Kotlin classes](https://blog.jetbrains.com/kotlin/2020/07/kotlin-1-4-m3-is-out-standard-library-changes/#fun-interfaces-in-stdlib)
* [Official - More on SAM conversions for Kotlin classes](https://blog.jetbrains.com/kotlin/2019/12/what-to-expect-in-kotlin-1-4-and-beyond/#sam-conversions)
* [Community request for the feature](https://youtrack.jetbrains.com/issue/KT-7770?_ga=2.103945408.1399009746.1597478338-1610252576.1591121905&_gac=1.256115449.1597482077.CjwKCAjwj975BRBUEiwA4whRByZeeWUfKevdonRWUAmo11kB3Q4ruIf4kMjpjkqWjrfJgr2GWA6nghoC47kQAvD_BwE)
* [Something about Kotlin Function Types](https://medium.com/tompee/idiomatic-kotlin-higher-order-functions-and-function-types-adb59172796).
* [Official Kotlin docs for SAM conversions](https://kotlinlang.org/docs/reference/java-interop.html#sam-conversions)
* [Details about Kotlin lambdas and SAM conversions](https://medium.com/tompee/idiomatic-kotlin-lambdas-and-sam-constructors-fe2075965bfb)
* [And a few more](https://medium.com/@krossovochkin/kotlin-java-interop-function-references-and-sam-conversions-3d0cd36f7967)

---

You might be interested in other posts I wrote about Kotlin, like:

* [Tail recursion and how to use it in Kotlin](https://jorgecastillo.dev/tail-recursion-and-how-to-use-it-in-kotlin)
* [Kotlin purity and function memoization](https://jorgecastillo.dev/kotlin-purity-and-function-memoization)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Kotlin posts 🙌
