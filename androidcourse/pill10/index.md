---
layout: page
current: androidcourse
title: Adding a screen to our app
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

There are different UI components that can represent a screen in an Android app. The most basic one is the **Activity**.

Any Android apps contain at least one Activity. Many of them have a single one and then implement some sort of navigation between different tabs inside. This is actually what Google recommends. But we'll get to that at the right time.

<img src="../../assets/images/activity.png" alt="My portrait pic" style="width:200px;">

> Activity is what you normally see on screen whenever you launch an app in your phone. **A fullscreen UI container**.

To add our first screen in the `app` module we must go to `app/src/main/java/`

But why `java` and not `kotlin`? 🧐 Because the `java` directory is **automatically created and marked as code sources by Android Studio**. Don't worry, we can add Kotlin files in there without any problem.

We could also configure an equivalent `kotlin` directory but that would require some extra Gradle configuration.

### Adding a package

The main mechanism to organize files within a module are the **packages**. You can think of them as folders containing code files. If you created the project using the Android Studio wizard you'll likely have a package already with the name you picked when creating the project. For me it is `dev.jorgecastillo.adoptme`.

In case you don't have a package, you can create it inside `app/src/main/java/` by doing `right click over java` -> `New` -> `package`, then write a package name.

> Remember that using a reverse domain name followed by the application name is a good practice.

### Adding the Activity

We have two options here. We can use the Android Studio wizard to create it for us or do it manually. If let Android Studio do it , it will create all the required files for us after filling some simple fields in a form.

> To do it we could `right click over the package` -> `New` -> `Activity` -> `Basic Activity` and we'd see the wizard like:

<img src="../../assets/images/add activity.png" alt="My portrait pic" style="width:400px;">

But I rather doing it manually 🙏 We are here to learn and that will give us a detailed overview of how it works.

So let's `right click over the package` -> `New` -> `Kotlin Class/File` -> then create a class named `LoginActivity` 👍 That will create the file for us with a `class`.

First thing we want to do is extending `AppCompatActivity`, which is a base class to provide support for basic theming and *fragments*. We'll get into fragments soon but think of them as subsections in our UI that we can easily reuse, add and replace. Useful for tabs for example.

```kotlin
class LoginActivity : AppCompatActivity() {
}
```

This won't do much as is. Note that **an Activity is a container for UI logic** often used to bind data to our UI, so we can display it on screen the way we want.

We want to add some UI to this Activity. We do that within the `onCreate` method that we can override from the parent. 🚨 Be careful! there are a couple of variants, and this is the one we want to override 👇

```kotlin
class LoginActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState) // Don't forget calling super!
        setContentView(R.layout.activity_main)
    }
}
```

This `onCreate` method is part of what it's called the Activity **"lifecycle"**.

Activities have a lifecycle with a few steps that the Android system calls in order. One of the initial steps is `onCreate`, that is called when the Activity is getting created. That's where the UI gets bound to the activity.

> As an optional extra bullet you can read the [Part I of this series](https://medium.com/androiddevelopers/the-android-lifecycle-cheat-sheet-part-i-single-activities-e49fd3d202ab) by [Jose Alcérreca](https://twitter.com/ppvi) where he describes the complete Activity lifecycle in detail 👍

🚨 Don't forget to call `super.onCreate(savedInstanceState)` since parent class needs to initialize a few things for properly supporting compatibility UI widgets, themes and other compatibility APIs among other things.

Then there is the `setContentView(R.layout.activity_main)` call. That is where we bind our UI logic (Activity) to the actual UI, which is represented with a **layout**.

We can see `activity_main` colored in red since we're still missing that file. We will learn how to create it (our first layout) in the next pill! 👍

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill10).

[Next: Creating our first layout >]({{ baseurl }}/androidcourse/pill11/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
