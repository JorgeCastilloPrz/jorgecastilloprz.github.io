---
layout: post
current: post
cover: assets/images/kyoto.jpeg
navigation: True
title: Digging into Kotlin Continuations
date: 2020-09-03 11:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Continuations represent the rest of a program. They are a form of control flow.

### ⏩ Continuation

A continuation is the implicit parameter that the Kotlin compiler passes to any `suspend` function when desugarizing. It is represented by an `interface`:

```kotlin
interface Continuation<in T> {
  abstract val context: CoroutineContext
  abstract fun resumeWith(result: Result<T>)
}
```

As you can see, it looks like a callback, because that is essentially what it is. You can also find it [within the official docs](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/).

A `Continuation` is **always linked to a coroutine**. Both terms are sometimes used indifferently, but there are important differences.

A coroutine has a `Continuation` associated, which stores the state of the coroutine and provides a couple of things:

* A `CoroutineContext` used to run it. It tells the coroutine **how to suspend** itself.
* A callback to wire results in the end. It uses the [Kotlin Result datatype](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) for that purpose, so you can resume your program either with a `success` or a `failure` as a result. Following docs:

> "It will resume the execution of the corresponding coroutine by passing the given value as the return value of the last suspension point."

This means the continuation represents the program flow. It decides how the program continues after some work, and that makes it another form of control flow.

There are also a couple of extension functions as shortcuts:

* `fun <T> Continuation<T>.resume(value: T)`
* `fun <T> Continuation<T>.resumeWithException(exception: Throwable)`

And a constructor function [in the standard library](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) to create a Continuation given a context and a callback function:

```kotlin
inline fun <T> Continuation(
    context: CoroutineContext,
    crossinline resumeWith: (Result<T>) -> Unit
): Continuation<T>
```

### 🤷‍♂️ But what does it represent?

Kotlin can `suspend` some work at a given point by starting a coroutine. That is be the afforementioned "suspension point". Here, you "suspend" your work and create a coroutine to perform some other work. Once there's a result of it, you can "resume" your work where you left it.

The `Continuation` is the way Kotlin remembers where the suspension points are and therefore where to return to. It also provides means to access information about **the state of the program right before the suspension point**. So it's literally a door to return the flow and the state of the program to the latest suspension point.

That makes it a control flow operator. It decides **how your program continues.**

### 🧮 What is it translated to

Let's say we've got a `suspend` function to perform some job we need to suspend, and a caller function like `main` in this case:

```kotlin
suspend fun doSomething() = "Done!"

suspend fun main() {
  doSomething()
}
```

If we remove the boilerplate and metadata to stay simple, that gets decompiled to something like this in Java (using the `tools` -> `Kotlin` -> `Show Kotlin Bytecode` IntellIJ menu option):

```java
// FileKt.java
public final class FileKt {
   @Nullable
   public static final Object doSomething(@NotNull Continuation $completion) {
      return "Done!";
   }

   @Nullable
   public static final Object main(@NotNull Continuation $completion) {
      Object var10000 = doSomething($completion);
      return var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED() ? var10000 : Unit.INSTANCE;
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      RunSuspendKt.runSuspend(new FileKt$$$main(var0));
   }
}
```

Obviating the need for an enclosing class given the lack of "package level" variables and functions in Java, the key point here is how both `suspend` functions have been converted to static functions that get the `Continuation` passed as an explicit argument, so it is used as a standard callback.

You can see how the `main` function needs to forward the continuation to the `doSomething($completion)` call.

All that is hidden to us by the Kotlin compiler, that allows to call `suspend` functions synchronously, while everything stays asynchronous under the hood. That's the magic and also the point of all this.

### 🕵️‍♀️ Java interoperability

Since our `suspend` functions are translated into public static functions within a class, we should be able to call them from java using a non sugarized standard callback style, by passing an instance of a `Continuation`:

```java
public class Main {

  public static void main(String[] args) {
    FileKt.doSomething(new Continuation<String>() {
      @Override
      public void resumeWith(@NotNull Object o) {
        // process result `o`
      }

      @NotNull
      @Override
      public CoroutineContext getContext() {
        return EmptyCoroutineContext.INSTANCE;
      }
    });
  }
}
```

The issue here is that starting on Kotlin 1.3, `Continuation` uses the [Kotlin Result](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) inline class for the `resumeWith` method, and **inline classes are not supported by Java**. If you want to use them you must provide explicit wrappers from Kotlin, since inline classes are reduced to its inner value by the Kotlin compiler, and types will not match for javac otherwise. More details on this [here](https://discuss.kotlinlang.org/t/inline-classes-tedious-to-use-considering-java-interop/9382).

One thing you can do is encoding your own `Continuation` implementation that maps the Kotlin result to something else, to get rid of it from its public api and open the usage from Java. We actually [do that in the Arrow library](https://github.com/arrow-kt/arrow-core/blob/master/arrow-core-data/src/main/kotlin/arrow/typeclasses/internal/Continuation.kt), but with a different purpose.

Another alternative is to translate it to a JDK 8 `CompletableFuture`, so the bridging allows you to keep the asynchronous use case covered:

```kotlin
fun doSomethingFromJava() =
  GlobalScope.future { doSomething() }

// Or with explicit CoroutineContext
fun doSomethingFromJava() =
  GlobalScope.future(Dispatchers.IO) { doSomething() }
```

That will require the jdk8 coroutines integration module though:
```groovy
dependencies {
  implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-jdk8'
}
```

And you'll be able to call it from Java:

```java
CompletableFuture<String> future = FileKt.doSomethingFromJava();
String result = future.get();
```

This is possible because the bridging is done on the Kotlin side, hence the `Result` type is not visible from Java anymore.

> Note that here we're using `GlobalScope` for the sake of the example, but you should likely use a narrower `CoroutineScope` that fits your requirements in case you are using KotlinX Coroutines library.

### 💡 Extra bullets

Continuations or Continuation Passing Style is a concept that you can find in Kotlin for encoding control flow, but you could take it further and implement "delimited continuations" based on a very similar idea. When a continuation represents the rest of a program, a delimited continuations capture only some part of it.

* [This gist](https://gist.github.com/sebfisch/2235780) by [Sebastian Fischer](https://gist.github.com/sebfisch) is an interesting source for starting to dig into the concept of Delimited Continuations.
* Also [this post](https://cs.ru.nl/~dfrumin/notes/delim.html) by [Dan Frumin](https://cs.ru.nl/~dfrumin/).

---

You might be interested in other posts I wrote about Kotlin, like:

* [Delimited continuations gist by Sebastian Fischer](https://gist.github.com/sebfisch/2235780)

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Kotlin posts 🙌
