---
layout: post
current: post
cover: assets/images/camera.jpeg
navigation: True
title: Sealed interfaces in Kotlin
date: 2021-03-06 10:00:00
tags: [kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Short overview of the sealed interfaces coming up in Kotlin 1.5.

## 🚨 Disclaimer

Sealed interfaces are **Experimental**. They may be dropped or changed at any time. You can give feedback on them [in YouTrack](https://youtrack.jetbrains.com/issue/KT-42433?_ga=2.63257064.633709735.1615016427-1586827560.1591541237).

## 🔎 Subclass location

Limitations on where to write the subclasses of a sealed class are a matter of compiler awareness. It needs to know about all the subclasses available in order to ensure exhaustiveness.

> Until not long ago, the compiler was not capable of looking further than the scope of the sealed class itself, so it was forbidden to declare subclasses outside of it. Kotlin 1.1 made it possible to declare those within the same file.

Starting on Kotlin 1.5 location restrictions will get relaxed, so we can declare them on different files **under the same module**. This restricts it to only implementations that "you own". The Kotlin compiler can still ensure exhaustiveness given that the module is compiled together. This is also possible for [sealed classes and sealed interfaces in Java 15](https://openjdk.java.net/jeps/360).

The aim is also to allow splitting large sealed class hierarchies into different files to make things more readable.

This will also go for sealed interfaces.

```kotlin
// Vehicle.kt
sealed interface Vehicle

// Cars.kt
object FuelCar : Vehicle
object ElectricCar : Vehicle

// Trains.kt
object HighSpeedRail : Vehicle
object MonoRail : Vehicle
object Tram : Vehicle
object InterCity : Vehicle

// Plane.kt
object Airliner : Vehicle
object Ultralight : Vehicle
```

Note that this change is also experimental. You can give feedback [here](https://youtrack.jetbrains.com/issue/KT-42433?_ga=2.134533430.633709735.1615016427-1586827560.1591541237).

## 🤔 Why not sealed class?

When we limit the implementations per module, our library can have public sealed interfaces as part of its API surface, therefore hiding the internal implementations of it and ensuring they'll not get extra implementations provided by the client ✅

That way both the library devs and the clients can leverage exhaustive evaluation over a contract represented by an interface.

But truth is you'd achieve the same with a sealed class, given they share the same limitation. So why to seal interfaces?

If we use `interfaces` across the board and there comes the need to limit the possible implementations of it, `sealed class` is not a valid replacement for all the cases.

One example of this would be enum classes that implement interfaces. [In Kotlin that is possible](https://kotlinlang.org/docs/enum-classes.html#implementing-interfaces-in-enum-classes). Given enums can't subclass other classes, a sealed class would not work.

It's also important to note that interfaces can implement multiple other interfaces, and sealed classes are limited to a single parent class, so there would be cases we cannot cover.

One interesting door we are opening with `sealed interface` is the fact that we can make a subclass be part of multiple sealed hierarchies.

Think of the following set of domain errors modeled with standard `sealed classes`:

```kotlin
sealed class CommonErrors // to reuse across hierarchies
object ServerError : CommonErrors()
object Forbidden : CommonErrors()
object Unauthorized : CommonErrors()

sealed class LoginErrors {
  data class InvalidUsername(val username: String) : LoginErrors()
  object InvalidPasswordFormat : LoginErrors()
  data class CommonError(val error: CommonErrors) : LoginErrors()
}

sealed class GetUserErrors {
  data class UserNotFound(val userId: String) : GetUserErrors()
  data class InvalidUserId(val userId: String) : GetUserErrors()
  data class CommonError(val error: CommonErrors) : GetUserErrors()
}
```

Let's imagine a couple of network requests to perform a login and to load the user details. Each request can produce some errors specific to its domain, but it could also yield one of the `CommonErrors` that are generic. With sealed classes, reusing those hierarchies becomes a bit dirty, since it requires adding an extra wrapper case to each hierarchy where we want to reuse it.

That creates a smell while processing it, since we are required to use nested when statements:

```kotlin
fun handleError(loginError: LoginErrors): String = when (loginError) {
  is LoginErrors.InvalidUsername -> TODO()
  LoginErrors.InvalidPasswordFormat -> TODO()
  is LoginErrors.CommonError -> when (loginError.error) {
    Forbidden -> TODO()
    ServerError -> TODO()
    Unauthorized -> TODO()
  }
}
```

This is far from ideal given we need to perform both checks for the outer and inner sealed classes separately.

Ideally we would want to flatten this by making the `CommonErrors` simply be part of both `GetUserErrors` and `LoginErrors` hierarchies. Problem is this is not possible with a sealed class because multiple inheritance is not a thing in Kotlin. But it is indeed possible with interfaces.

```kotlin
sealed interface CommonErrors
object ServerError : CommonErrors, GetUserErrors, LoginErrors
object Forbidden : CommonErrors, GetUserErrors, LoginErrors
object Unauthorized : CommonErrors, GetUserErrors, LoginErrors

sealed interface GetUserErrors
data class UserNotFound(val userId: String) : GetUserErrors
data class InvalidUserId(val userId: String) : GetUserErrors

sealed interface LoginErrors
data class InvalidUsername(val username: String) : LoginErrors
object InvalidPasswordFormat : LoginErrors
```

Given a class or object can implement as many interfaces as we want, we can make `ServerError`, `Forbidden`, and `Unauthorized` be part of both sealed hierarchies -- `GetUserErrors` and `LoginErrors`.

And now we're enforced to cover all the cases on a `when` expression without any nesting:

```kotlin
fun handleError(error: LoginErrors): String = when (error) {
  is InvalidUsername -> TODO()
  InvalidPasswordFormat -> TODO()
  ServerError -> TODO()
  Forbidden -> TODO()
  Unauthorized -> TODO()
}

```

It requires each implementation to declare the hierarchies it implements, so in cases we have an error that needs to be part of lots of hierarchies things could get a bit dirty, but it is coherent with how subclasses of sealed classes need to declare what hierarchy they belong to.

For deeper reasoning about why to introduce the concept of sealed interfaces in the language you can read [the original proposal](https://github.com/Kotlin/KEEP/blob/master/proposals/sealed-interface-freedom.md).

## How to try it 👇

You'll need Kotlin 1.5.

```groovy
tasks.withType<KotlinCompile> { // In Groovy: compileKotlin {
    kotlinOptions {
        languageVersion = "1.5"
        apiVersion = "1.5"
    }
}
```

---

You might be interested in other Kotlin posts I wrote:

* [Tracking side effects with suspend](https://jorgecastillo.dev/tracking-side-effects-with-suspend)
* [Kotlin Continuations](https://jorgecastillo.dev/digging-into-kotlin-continuations)
* [Kotlin SAM in 1.4](https://jorgecastillo.dev/kotlin-sam-conversions)

This post with the proposal for introducing sealed classes and sealed interfaces in Java 15 was also interesting to me:

* [Java 15 Proposal for sealed classes and interfaces](https://openjdk.java.net/jeps/360)

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

More interesting stuff to come 🙌
