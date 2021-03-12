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

The application block has some default things like whether it uses the default automatic backup system by Android, a reference to the app icon, the label to use for the app name, whether it supports RTL (right to left) and a reference to the theme. These attributes will be described during the course.

### Adding our Activity

All Activities need to be registered on the manifest or the app will not compile.

The main activity is not different, and it has to be declared as the "MAIN" and "LAUNCHER" one like this 👇

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

I'm omitting the attributes on the `manifest` and `application` tags to make it more readable since we have already seen those.

See how we're referencing the Activity with a path relative to the package name. In this case it's in the root of the package.

Also note how we can pick a label for the activity so it shows by default in the top bar. We are referencing the app name for it from a string resource. This is how we store texts in Android. You can cmd (or ctrl) + click on it to navigate and see its content.

The most important thing here is the `intent-filter`

### Our first Intent filter

Android uses a system based on "intent filters" to know how to treat an app. Intents reflect actions that can be used to launch an app, a service, broadcast a message, and more. There is a lot of literature about intents and we'll learn about many, but we're only interested on this one for now.

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

This filter states that the `LoginActivity` will be the main entry point to this app, and that it needs to be listed in the Android Launcher, so we can see the app icon listed with any other apps installed on the device.

You can find much more about intent filters [here](https://developer.android.com/guide/components/intents-filters), but I recommend you leave that for later. Let's learn step by step.

Once we have that we can go ahead and click the play icon one more time. It might take some time but we'll get the app built, compiled and deployed (that is installed) on the device. It will also launch automatically 🚀

<img src="../../assets/images/our first app.png" alt="Android Studio" style="width:300px;">

And here we have our blank screen. **Congratulations!** 🎊 You have created your first Android app. See how a top bar is added by default and it's displaying the app name we've set in the manifest.

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill13).

[Next: Asd >]({{ baseurl }}/androidcourse/pill14/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
