---
layout: page
current: androidcourse
title: Adding the Activity to the AndroidManifest
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

### AndroidManifest

It is a basic XML file with metadata that Android needs to understand how to launch the app and some other requirements of it. The default version of it was generated in `app/src/main/AndroidManifest.xml` and looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="dev.jorgecastillo.adoptme">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AdoptMe" />

</manifest>
```

The `xmlns:android` attribute is the Android namespace so we can reference `android:` attributes like the ones we see below in the `application` tag.

The package name we picked when creating the project is also there.

The application block has some default settings like whether it uses the default automatic backup system by Android, a reference to the app icon, the label to use for the app name, whether it supports RTL (right to left) and a reference to the theme. These attributes will be described during the course.

### Registering our Activity

All Activities need to be registered on the manifest or the app will not compile.

The main activity is not different, and it has to be declared as the "MAIN" and "LAUNCHER" one inside the `application` tag like this 👇

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ...>

    <application ...>

        <activity
            android:name=".LoginActivity"
            android:label="@string/app_name"
            >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

> I'm omitting the attributes on the `manifest` and `application` tags to make it more readable since we have already seen those.

See how we're referencing the Activity with a path relative to the package name. In this case it's in the root of the package, so `.LoginActivity` it is.

Also note how we can pick a label for the activity so it shows by default in the top bar. We are referencing the app name for it from a string resource. This is how we store texts in Android. You can cmd (or ctrl) + click on it to navigate and see its content.

And finally the most important thing here: The `intent-filter` 🤔

### Brief intro to Intents

Android uses a system based on the so called "Intents" to trigger actions. These actions can be performed by the same app **or a different one**.

Intents can be **explicit**, where we declare the exact application or component that will satisfy the intent. These are frequently used to perform actions within our own app, like starting activities (navigation), broadcasting messages, starting services, or similar things.

Intents can also be **implicit**. These don't name a specific component but declare a general action to be performed. These **allow other apps to handle it**. This is where "Intent filters" come into play.

### Our first Intent filter

Intent filters allow us to expose apps as ready to handle a given category of intent. If multiple apps expose themselves as ready to handle the same category, all of them will appear as alternatives to pick from in the menu whenever we trigger the intent.

> Remember that share sheet in Android where you need pick an app from a list?

There is a lot of literature about intents and we'll learn about many, but we're only interested on this one for now 👇

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

An Intent carries an **action** and a **category** among other things. The action stands for the action to perform. The category is used to infer what components will be able to handle this intent.

This intent filter will filter any intents with an action of type `MAIN` and category `LAUNCHER`.

* The `MAIN` action means that the `LoginActivity` will be **the main entry point to this app**.
* The `LAUNCHER` category means that this activity has to be listed in the Android Launcher so we see the app icon listed with any other apps installed on the device.

The Android Launcher is an application by itself. It triggers an Intent to detect all apps with activities filtering the `LAUNCHER` category **to know which ones to list**.

You can find much more about intent filters [here](https://developer.android.com/guide/components/intents-filters), but I recommend you leave that for later. Let's learn step by step.

Once we have that we can go ahead and click the play icon one more time. It might take some time but we'll get the app built, compiled and deployed (that is installed) on the device. It will also launch automatically 🚀

<img src="../../assets/images/our first app.png" alt="Android Studio" style="width:300px;">

And here we have our blank screen. **Congratulations!** 🎊 You have created your first Android app. See how a top bar is added by default and it's displaying the app name we've set in the manifest.

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill13).

[Next: Adding a text >]({{ baseurl }}/androidcourse/pill14/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
