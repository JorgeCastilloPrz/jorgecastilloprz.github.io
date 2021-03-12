---
layout: page
current: androidcourse
title: Running our app
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

If we look at the topmost menu bar of Android Studio we will find the folling options close to the project name.

<img src="../../assets/images/empty run configurations.png" alt="Android Studio" style="width:400px;">

The hammer compiles the project without running it. I'd recommend to compile the project frequently to detect compile time errors in our codebase.

The first dropdown is for the Gradle module we want to build and run. It needs to be an Android application module (applies the `com.android.application` plugin on it's `build.gradle`). In this case the `app` module.

The second dropdown is for the **Android emulator**, which we'll describe in a second.

And the play icon ▶️ is meant to build and run the app, as you probably imagined.

### 2 possible targets

Android apps can be run in two potential targets:

* 💻 An Android emulator in our computer.
* 📱 A physical (real) device.

### Android Emulator

Android runs on a Java virtual machine which means it cannot run natively on a computer. To emulate an Android device on our machine we have the emulator. Per the official docs:

> "It simulates Android devices on your computer so that you can test your application on a variety of devices and Android API levels without needing to have each physical device."

To create an emulator you can open the AVD Manager from the second dropdown. "AVD" stands for "Android Virtual Device", which is just another way to call the emulator.

By opening it we'll see a wizard to create our first Android Emulator. During the process we will need to pick the name, the device (since each device has a different screen ratio), the OS version to emulate, and a few more settings.

<img src="../../assets/images/create emulator 1.png" alt="Android Studio" style="width:600px;">

<img src="../../assets/images/create emulator 2.png" alt="Android Studio" style="width:600px;">

<img src="../../assets/images/create emulator 3.png" alt="Android Studio" style="width:600px;">

The process is simple, but [here you have the official documentation on how to create AVDs](https://developer.android.com/studio/run/managing-avds) in case you want to dig deeper.

Once you complete the process you'll have your emulator selected in the dropdown.

<img src="../../assets/images/run configurations.png" alt="Android Studio" style="width:600px;">

So the final thing to do is clicking the play icon and waiting. That will build the project and deploy it to the emulator (which means install it). It will also launch the app as soon as it's ready. Keep in mind the more complex the project is the longer it will take.

### Physical device

For running our app on a real device, we can enable the developer options on it and then connect it. Once we are done it will also display in the dropdown, so we can select it to run our app. We'll be able to switch between real devices and emulators at will.

It's possible to connect a real device to the computer both by cable or over Wi-Fi. The setup is pretty straightforward for both and well described [here](https://developer.android.com/studio/run/device).

### Running the app

Let's select the emulator we just created and click the play icon ▶️

The emulator will launch but **the app will not**! we'll get the following error 💥

> Could not identify launch activity: Default Activity not found
Error while Launching activity

This means we need to register our Activity in the `AndroidManifest.xml`. You will find this file in `app/src/main/AndroidManifest.xml`.

---

The emulator is ready but there's a final step before we can run our app. Learn about it on the next pill 💊

[Next: Adding the Activity to the AndroidManifest >]({{ baseurl }}/androidcourse/pill13/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
