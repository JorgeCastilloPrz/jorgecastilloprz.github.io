---
layout: page
current: androidcourse
title: Project structure - build.gradle
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

The root `build.gradle` is meant to fit configuration options **common to all Gradle modules** in the project. You will always find a root `build.gradle` on any Android app.

This is what we can find inside so far 👇

```groovy
buildscript {
    ext.kotlin_version = "1.4.31"
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.2"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

allprojects {
    repositories {
        google()
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

Let's talk a bit about the different blocks briefly.

#### buildscript

```groovy
buildscript {
    ext.kotlin_version = "1.4.31"
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.2"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```

This block is used to set **external dependencies for all the build scripts**, referring to "build scripts" as the `build.gradle` files for all modules. The dependencies will be available from all of them (e.g: `app/build.gradle`).

We can use this block to include relevant **plugins** in the build script classpath so Gradle can use them for building any module.

Given we want to write an Android app using Kotlin, we will add two plugins:

* **Android gradle plugin** so all modules can understand and build Android. It adds several features that are specific to building Android apps. It is **required to build Android projects using Gradle**. This plugin is typically updated in lock-step with Android Studio, so you'll end up updating it for each Android Studio update.
* **Kotlin gradle plugin** so all modules can understand Kotlin (indexing, autocomplete, highlight, compile, etc).

> Note that the plugin versions might vary by the time you consume this course.

Gradle searches for the plugins in the repositories listed under the `repositories` block, in order, when the time comes.

You can also use the buildscript to define some project wide external variables with the syntax:

```groovy
ext.kotlin_version = "1.4.31"
```

That will make those variables available in all the `build.gradle` scripts in the project so we can reuse them everywhere. This is a good practice I recommend to avoid ending up with multiple versions of the same library in the project 🚨

#### allprojects

Meant to configure this project (the root one) and all the sub-projects (modules like `app`).

```groovy
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```

Here we are setting a list of repositories to search dependencies in for all the projects. We could even configure the dependencies here if we wanted, in case those need to be shared:

```groovy
allprojects {
    repositories {
        google()
        jcenter()
    }
    dependencies {
      // ... we'd list them here
    }
}
```

There is much more we can do here but we'll leave it for now to keep it simple.

There is also a `subprojects` alternative block (instead of `allprojects`). That would only be applied to all the gradle modules (also known as sub-projects) and **not to the root one**. It's important to notice the difference.

#### clean task

<img src="../../assets/images/gradle clean 2.png" alt="Android Studio" style="width:400px;">

Gradle allows to define tasks that can be run either by command line or GUI. You can think of them as mini-scripts that can be run independently or as one of the steps in a build.

For GUI you can use the green "play" icon next to the task definition (see the image above), or also the Gradle panel on the right side:

<img src="../../assets/images/gradle panel.png" alt="Android Studio" style="width:400px;">

You can find the "clean" task there. Otherwise you can open the **terminal** tab at the bottom and type `./gradlew clean` from the root directory. All of them are different ways to run the same gradle task.

<img src="../../assets/images/gradle clean.png" alt="Android Studio" style="width:800px;">

> `gradlew` is the Gradle wrapper script we described in previous pills 👍

This clean task **deletes the build directory**. That is sometimes needed when we want to enforce the build files to get generated again for a new build.

This can help us to fix some issues coming from a previous build that might stick around otherwise.

Gradle allows to define as many tasks as we want, and also provides lots of default ones we will learn gradually over the course. We can search them in the Gradle panel mentioned above.

---

We already have our common gradle config for all modules ready ✅ Let's learn about the gradle config per module now.

[Next: Project Structure - app/build.gradle >]({{ baseurl }}/androidcourse/pill7/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
