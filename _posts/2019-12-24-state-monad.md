---
layout: post
current: post
cover: assets/images/monteagudo3.jpeg
navigation: True
title: Making state changes explicit with the State Monad
date: 2019-12-24 14:00:00
tags: [fp, kotlin]
class: post-template
subclass: 'post'
author: jorge
---

The astounding Cristo de Monteagudo, very close to my home in Murcia 😲. Let's discover how to model our state changes in an explicit way in our functional programs.

> This post is an intro to the `State` Monad. If you are already familiar with it you can jump to [the 2nd part I wrote in the 47 Degrees blog](https://www.47deg.com/blog/conway-kotlin/) that focuses on `StateT`.

### Problem?

Usually we find programs that encode state changes in an implicit way, what makes it harder to reason about those in the context of a complete program. This can promote unintended state updates as the result of unrelated operations (side effects).

We'll understand this better with an example. Here we have a bank account that has an `id`, an account `number`, and a `balance` that is going to be initially mutable for the sake of example. We will address mutability later.

```kotlin
data class Account(val id: String, val number: Long, var balance: Double)
```

Let's declare a `Bank` that will provide operations to get the balance of, withdraw from, or deposit money to a given account.

```kotlin
data class Bank(val name: String, val swiftCode: String, val accounts: List<Account> = listOf()) {

    fun getAccountBalance(accountId: String): Double =
        accounts.find { it.id == accountId }?.balance ?: 0.0

    fun withdraw(accountId: String, amount: Double): Double {
        val account = accounts.find { it.id == accountId }
        return if (account != null && account.balance >= amount) {
            account.balance -= amount
            amount
        } else {
            0.0
        }
    }

    fun deposit(accountId: String, amount: Double): Double {
        val account = accounts.find { it.id == accountId }
        return if (account != null) {
            account.balance += amount
            amount
        } else {
            0.0
        }
    }
}
```

This bank will represent our program state. Since our domain logic just requires to update account balances that's the only piece we made mutable. The bank has an initial list of accounts that we default to empty for simplicity.

Note how all operations return `Double`. That stands for the balance of an account, or the amount withdrawn or deposited, respectively. It'll be `0.0` if the account is not found for any of the three operations, or in case we're withdrawing and there's not enough balance on it for the passed amount.

Let's say we want to encode an operation to move half of the balance of one account to another. It will find an account by id and check its balance, then it will withdraw half of it and deposit the withdrawn money into another account. We could encode it like this:

```kotlin
fun transferHalfOfTheMoney(bank: Bank, from: String, to: String): Double {
    val balance = bank.getAccountBalance(from)
    val withdrawn = bank.withdraw(from, balance / 2)
    return bank.deposit(to, withdrawn)
}
```

In case any step returns `0.0` the whole computation will end up returning `0.0` given how the operations are written. We could finally run our program like this.

```kotlin
fun main() {
    val bank = Bank(name = "Central Bank", swiftCode = "CBESMMX")
    val transferred = transferHalfOfTheMoney(bank, from = "101931", to = "101932")
    println(transferred)
}
```

This encoding is acceptable for an OOP style, but it has a problem: **State changes are implicit**. Each one of our functions needs to access and modify a shared state, (the `accounts` balance) that is external to the scope of the function itself. Let's improve that a little bit.

### Avoiding implicit state changes

Getting back to the essence of Functional Programming, functions would expect their **dependencies to be passed as arguments** so they can operate with their inputs to provide a result.

For this example we can refactor our functions to receive the shared state (`Bank`) as an argument and return the **new** updated state as a result. Since we still also need the previous function result as it was, we can return a tuple containing both things:

```kotlin
data class Bank(val name: String, val swiftCode: String, val accounts: List<Account> = listOf())

fun getAccountBalance(bank: Bank, accountId: String): Tuple2<Bank, Double> =
    Tuple2(bank, bank.accounts.find { it.id == accountId }?.balance ?: 0.0)

fun withdraw(bank: Bank, accountId: String, amount: Double): Tuple2<Bank, Double> {
    val account = bank.accounts.find { it.id == accountId }
    return Tuple2(bank, if (account != null && account.balance >= amount) {
        account.balance -= amount
        amount
    } else {
        0.0
    })
}

fun deposit(bank: Bank, accountId: String, amount: Double): Tuple2<Bank, Double> {
    val account = bank.accounts.find { it.id == accountId }
    return Tuple2(bank, if (account != null) {
        account.balance += amount
        amount
    } else {
        0.0
    })
}
```

Since we retain the `bank` reference, it will be updated accordingly and returned.

Note how this encoding **removes the need for a wrapper class** since now we can pass our `Bank` as a value object. In Functional Programming, data is passed along using immutable value objects, and data types provide surrounding context that determines the way they are operated.

But there is still an big issue with this encoding: **Mutability**. Our functions get their dependencies as input arguments and provide the new state as a result, but they are still modifying the passed in state implicitly, which is still a side effect. So this refactor wasn't really a big win at all.

Let's remove mutability then, it should be easier given our `Bank` is now a value object. Since the only mutable piece in the whole thing was the balance, we can update our `Account` to be completely immutable.

```kotlin
data class Account(val id: String, val number: Long, val balance: Double)
```

That means every time we need to update an account in our state, or even our overall state (bank), we will need to create new instances, since both are value objects.

We will also need to refactor our functions to match the updated structure. We can use the `copy` function to create the required new instances:

```kotlin
fun getAccountBalance(bank: Bank, accountId: String): Tuple2<Bank, Double> =
    Tuple2(bank, bank.accounts.find { it.id == accountId }?.balance ?: 0.0)

fun withdraw(bank: Bank, accountId: String, amount: Double): Tuple2<Bank, Double> {
    val account = bank.accounts.find { it.id == accountId }
    return if (account != null && account.balance >= amount) {
        Tuple2(
            bank.copy(accounts = bank.accounts.map { acc ->
                if (acc.id == accountId) {
                    acc.copy(balance = acc.balance - amount)
                } else {
                    acc
                }
            }),
            amount
        )
    } else {
        Tuple2(bank, 0.0)
    }
}

fun deposit(bank: Bank, accountId: String, amount: Double): Tuple2<Bank, Double> {
    val account = bank.accounts.find { it.id == accountId }
    return if (account != null) {
        Tuple2(
            bank.copy(accounts = bank.accounts.map { acc ->
                if (acc.id == accountId) {
                    acc.copy(balance = acc.balance + amount)
                } else {
                    acc
                }
            }),
            amount
        )
    } else {
        Tuple2(bank, 0.0)
    }
}
```

It's probably a bit more verbose now but mutability is definitely worth it. We could improve it a bit by using **Optics** to simplify work with nested immutable structures, but we will leave that for a future post.

Even if our state is mutable by nature (all apps have evolving mutable state), we'll face less raise conditions, side effects and other problems derived from mutability when our state can be represented as an immutable structure (and passed around without fear).

So now that we've got all of our functions working with immutable data, if we step back for a second we will find that all our functions have the following type declaration:

`(Bank) -> Tuple2<Bank, Double>`

This exact encoding can be simplified a bit more using the data types provided by Arrow.

### Explicit state management

[We've got](https://arrow-kt.io/docs/arrow/mtl/state/) the `State` data type to model state changes in functional programming.

`State` wraps a computation with the type `(S) -> Tuple2(S, A)`, where `S` corresponds to the input and output states, and `A` is a potential intermediate result of the state update. This can directly replace the encoding we achieved for our program.

> Note that **`State` models state updates, not state** by itself. You're free to pass any structure for your state using the `S` type, even a mutable one. State by itself doesn't ensure immutability though, but it helps on making state changes explicit so it becomes easier to reason about how it evolves across the program, and also to implement it using immutable structures (recommended).

Let's update our code to make it reflect state changes **on its type**.

```kotlin
fun getAccountBalance(accountId: String) = State<Bank, Double> { bank ->
    Tuple2(bank, bank.accounts.find { it.id == accountId }?.balance ?: 0.0)
}

fun withdraw(accountId: String, amount: Double) = State<Bank, Double> { bank ->
    val account = bank.accounts.find { it.id == accountId }
    if (account != null && account.balance >= amount) {
        Tuple2(
            bank.copy(accounts = bank.accounts.map { acc ->
                if (acc.id == accountId) {
                    acc.copy(balance = acc.balance - amount)
                } else {
                    acc
                }
            }),
            amount
        )
    } else {
        Tuple2(bank, 0.0)
    }
}

fun deposit(accountId: String, amount: Double) = State<Bank, Double> { bank ->
    val account = bank.accounts.find { it.id == accountId }
    if (account != null) {
        Tuple2(
            bank.copy(accounts = bank.accounts.map { acc ->
                if (acc.id == accountId) {
                    acc.copy(balance = acc.balance + amount)
                } else {
                    acc
                }
            }),
            amount
        )
    } else {
        Tuple2(bank, 0.0)
    }
}
```

Done! Our functions return `State<Bank, Double>` now. This defines our functions as a state change imposed by `State`, which implies the following things:

- `State` helps to manage the state of a program effectively and leverages the usage of immutability (if you want to).
- `State` passes the state along by itself. Note how **we don't need to pass it as an argument anymore**.
- Updating state implicitly implies side effects since the functions to update it need to access the state which is external to the scope of the function.
- Our state always changes sequentially by nature. Modelling that becomes as easy as chaining different `State`s using `flatMap`.
- Both `map` and `flatMap` work over the type constructor `A`, which is `Double` in our example. That means you're able to switch `A` to a different type on some arbitrary state changes if you want to, so you can vary intermediate results at will. But `S` is imposed and **invariant** for the whole call chain. Following that, the state will be passed along the way so you don't need to do it manually. This encoding ensures sequentiality in the way that the input state for a state change will always be the result of the previous one, enforcing by how `flatMap` is encoded.
- `State` provides a function to run it providing an initial value for it.

Let's refactor our `transferHalfOfTheMoney` use case now.

```kotlin
fun transferHalfOfTheMoney(from: String, to: String): StateT<ForId, Bank, Double> =
    State.fx(idMonad) {
        val balance = getAccountBalance(from).bind()
        val withdrawn = withdraw(from, balance / 2).bind()
        deposit(to, withdrawn).bind()
    }
```

There are some things to highlight on this snippet:

* We are returning `StateT<F, S, A>` now, instead of `State<S, A>`. We will learn what `StateT` stands for in a second so please don't pay much attention for now. Just note how the function is now also declared as a `State` change in our program.
* We are using [ArrowFx](https://arrow-kt.io/docs/fx/) for encoding our function. Note that our operations are sequential. The input of each operation is the result of the previous one. That means `flatMap`, but instead, we can use the *Monad Comprehensions* style by ArrowFx. Here, `bind()` stands for `flatMap` under the hood, but it helps to encode our program using a synchronous-like syntax, what makes it more readable.

Finally, we would run our program using the `run` function that requires passing the initial state:

```kotlin
fun main() {
    val bank = Bank(name = "Central Bank", swiftCode = "CBESMMX")
    val transferred = transferHalfOfTheMoney(from = "101931", to = "101932")
      .run(idMonad, bank)

    println(transferred)
}
```

And this would wrap up our program with a pretty valid functional encoding. But we can still move one more step forward 👍

### Interleaving side effects

We always need some side effects in our programs besides the state management, so we'd ideally want to use data types that ensure side effect control and error handling like `IO`, or `BIO`. The problem is different *"Monads do not compose"* as in: Different data types don't really play well together. If we wanted to mix `IO` and `State` that would result in a lot of bridging between both data types. Our code would become barely readable.

We've got `StateT`, aka: *State Transformer* to help on this. It provides state handling capabilities to any `F` data type, so it provides ergonomics to solve the mentioned problem.

`StateT<ForIO, Bank, Double>` is the type we were returning from our `State.fx` block above. In that case `F` was `Id`, which is the *identity monad*, which represents **no effect** (given we didn't need any special treatment for our operations). Feel free to take a look to [the official docs](https://arrow-kt.io/docs/apidocs/arrow-core-data/arrow.core/-id/index.html#id) to understand better. That is why we needed to pass in an instance of `IdMonad` in a couple of places.

For a complete blogpost on how to update our programs to use `StateT` I recommend [this blogpost I wrote for 47 Degrees](https://www.47deg.com/blog/conway-kotlin/) where we solve the Conway's Game of Life kata using `StateT` and `IO` as the `F` for logging state. Take it as **the second part of this series**.

### Some links

* [Arrow's State docs](https://arrow-kt.io/docs/arrow/mtl/state/)
* [Arrow's StateT docs](https://arrow-kt.io/docs/arrow/mtl/statet/)
* [ArrowFx docs](https://arrow-kt.io/docs/fx/)
* [Second part about StateT](https://www.47deg.com/blog/conway-kotlin/)

If you are interested in Functional Programming in Kotlin and Arrow, I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). Don't hesitate to follow! 🙏🏽

See you soon and happy Christmas in case you are celebrating! 👋🎄🎊

### Want to support me?

If you reached this point you might consider supporting me for boosting my will to write. If that's the case, here you have a button, really appreciated! 🤗

Supported or not, I will keep writing and providing content for free ✅

<a href="https://www.buymeacoffee.com/jorgecastillo" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" style="height: 51px !important;width: 217px !important;" ></a>
