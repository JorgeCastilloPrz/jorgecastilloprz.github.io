---
layout: page
current: androidcourse
title: Android configuration
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

These initial posts might be too theoretical, but we'll find them useful to have a bird-eye perspective of the structure of a project before we start adding content.

Believe me, we'll be done before you can notice! 🙏

Time to go over the `android` block within the `app/build.gradle` build script.

### android

Here is where we fit all the Android related configuration for this module. We'll find something like this on every Android app out there. 👇

```groovy
android {
    compileSdkVersion 30
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "dev.jorgecastillo.adoptme"
        minSdkVersion 21
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}
```

* `compileSdkVersion`: The Android API level Gradle should use to compile our app. This means our app can use the API features included in this API level and lower. `30` stands for Android 11.
* `buildToolsVersion`: The version of the SDK build tools, command-line utilities, and compiler that Gradle should use to build our app.

### defaultConfig

This is the default configuration for all the variants of the app we can build. We will learn more about the variants in a second.

* `applicationId`:  Uniquely identifies the app on the device and in the Google Play Store. Matches the package name by default, but they are independent.
* `minSdkVersion`: The minimum supported API version. Our app will not be able to get installed in older versions.
* `targetSdkVersion`: Indicates that we have tested against the target version and the system should not enable any compatibility behaviors to maintain the app's forward-compatibility with the target version.
* `versionCode` and `versionName` describe the version of the app. Will need to be updated incrementally each time we want to publish a new version.
* `testInstrumentationRunner` used to run Android tests. Not relevant yet.

### buildTypes

We can think of build types as different ways to build the same app. We reuse the same sources (code), but **configure the build differently**.

They are used to specify some settings that vary per variant, like initializing a global variable with the url for the server environment (develop, staging, production...), or things like whether we want to enable minify for the variant (code shrinking, obfuscation, and optimization), for example.

There is an implicit one called `debug`, and Android Studio adds a `release` one when creating the project. Those are the default ones, but we could also add a `staging` one, for example, or whatever else. **We can call them whatever we want.**

The `debug` buildType is implicitly configured with `debuggable true` by the Android Gradle Plugin, which makes us able to **debug the application during development**. It is set to `false` by default for any other buildTypes for security reasons.

The debug type also gets a default signing configured so the application is signed using a default debug certificate which is common for all Android apps.

> Note that Android apps need to be signed before deploying them to the device or uploading them to the Play Store.

We could also add the `debug` type explicitly along with the `release` one in case we needed to explicitly configure any properties for it. Let's not do it for now.

"Minify" (code shrinking, obfuscation, and optimization) is disabled by default for the `release` type. Let's keep it like that for now.

```groovy
buildTypes {
    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

The `proguardFiles` Includes the default ProGuard rules files that are packaged with the Android Gradle plugin.

There is a tool called **R8** in Android that basically converts the Java bytecode into the DEX format that runs on the Android platform, and takes the chance to perform the mentioned shrinking, obfuscation, and optimization among other things.

R8 uses ProGuard rules files to modify its default behavior and better understand your app’s structure, such as the classes that serve as entry points into your app’s code.

### compileOptions

```groovy
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
```

Since R8 converts Java bytecode to DEX we need to give it a clue on what Java version to support. Here we are picking Java 8 which has been the recommended one so far, but note that soon Java 11 will be required. That is because Android supports a subset of the features of Java versions newer than 7.

### kotlinOptions

```groovy
kotlinOptions {
    jvmTarget = '1.8'
}
```

Kotlin also needs to know what Java version we are targeting, given Kotlin is decompiled to Java compatible bytecode for interoperability between both languages.

Android has been Java dependant since ever and this is the price to pay for a language to work fine in the platform.

---

And with this we've got an idea on how the build is configured for an Android module ✅ Time to learn about the `dependencies` block.

[Next: Project Structure - App dependencies >]({{ baseurl }}/androidcourse/pill9/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
