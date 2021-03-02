---
layout: page
current: androidcourse
title: Our first project
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

We already got Android Studio installed. Time to create our first Android project! 🙌

Most of us love pets so we are building a pet adoption app 🐶 Let's pick the option to "Create New Project".

<img src="../../assets/images/android studio main menu.png" alt="Android Studio" style="width:600px;">

Then select "No Activity" from the Phone and Tablet tab, as the template to use.

<img src="../../assets/images/android studio create project 1.png" alt="Android Studio" style="width:600px;">

We will use this template as a starting point since it provides an empty application structure, so we can start adding our screens on top of it 👍

We will get back to it in detail, but for now we can think of an "Activity" as a screen within our app.

Click on "Next" and we will get a form for some **initial project configuration**.

<img src="../../assets/images/android studio project config.png" alt="Android Studio" style="width:600px;">

What we have here:

* **Name:** For the project. The app name displayed on the device (after installing) is defaulted to this one, but can be changed independently.
* **Package name:** Highly important. It identifies our app once uploaded to the [Google Play Store](https://play.google.com/store/apps) (the app store). It's a good practice to use a reversed domain name for it, even if not required. In my case I own [jorgecastillo.dev](https://jorgecastillo.dev), so `dev.jorgecastillo` followed by the app name seems fine.
* **Save location:** The directory to save the project while we work on it.
* **Language:** Let's pick Kotlin since it's the [official programming language for Android](https://developer.android.com/kotlin).
* **Minimum SDK:** The minimum version of the Android OS we wan to support. The lower it is, the more Android devices our app will be able to be installed on. We will get back to this one in a second.
* **Use legacy android.support libraries:** Keep this one unchecked since that's not recommended anymore.

### Minimum SDK - A small disclaimer 🚨

If we look again to the image above, we'll notice that the wizard already gives us some info about the percentage of devices that will support our app depending on the Minimum SDK we choose.

<img src="../../assets/images/android studio project config 2.png" alt="Android Studio" style="width:600px;">

The minimum allowed here is Android 4.1, which is **super old (July 2012)**. If we picked that one, we'd support a 99.8% of the devices out there, so it would seem like a great pick at a first glance.

The issue with this is that by picking a too old min SDK, we would require lots of conditional logics in our code to make it both retrocompatible and still support the newer nicer features only available on newer versions of the SDK. Per my experience, this can become quite counterproductive 😔

I highly recommend **finding a good compromise between the amount of devices supported and the code quality and maintenance**.

Android 5.0 (Lollipop) is the recommended pick nowadays, since more than a 94% of devices are supported, which gives a very good range, and APIs started getting much better on that version.

Also note that older versions are quite outdated and the inmense majority of the device manufacturers already updated their system version over 5.0. Latest released Android version is Android 11, with Android 12 coming up this year, so we can figure.

The Android Studio wizard also provides more detailed information about the distribution charts per Android versions. Click on "Help me choose" right below the picker, and we'll see this awesome diagram:

<img src="../../assets/images/android distribution.png" alt="Android Studio" style="width:600px;">

So once we pick **API 21: Android 5.0 (Lollipop)** for the Minimum SDK, we can click on "Finish" and get ready to start coding our initial Android app 🎉

<img src="../../assets/images/androidstudio.jpg" alt="Android Studio" style="width:600px;">

[Next: Project Structure - Overview >]({{ baseurl }}/androidcourse/pill3/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
