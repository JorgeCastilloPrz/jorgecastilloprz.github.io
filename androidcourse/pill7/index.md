---
layout: page
current: androidcourse
title: Application build.gradle
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

If we expand the `app` module directory we'll find another `build.gradle` script 🤔

<img src="../../assets/images/app module structure.png" alt="My portrait pic" style="width:400px;">

That one contains the configuration specific to this gradle module. Let's have a look at it, since there are some things important to highlight.

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}

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

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.3.2'
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.3.0'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
```

Let's go over the different blocks one by one.

#### plugins

Here we can add plugins that will apply only to this module.

If we remember from previous pill we imported a couple of plugins into the classpath using the `buildscript` block:

```groovy
buildscript {
  ...
  dependencies {
    classpath "com.android.tools.build:gradle:4.1.2"
    classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
  }
}
```

This means those plugins become accessible from all the modules and ready to use, but **each module needs to proactively activate them** if needed.

That is what we are doing in the `plugins` block 👇

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}
```

**com.android.application**

The Android application plugin is part of the Android Gradle Plugin and enables the `android` configuration block which is where we'll configure all our Android-specific build options in a second. We could say that it "enables Android development" for this module.

There is an alternative `com.android.library` we would use instead if we were creating an Android library, which is not the case for this project.

**kotlin-android**

This plugin enables the Kotlin language for an Android module.

Any Android module app written in Kotlin will require at least these two plugins.

---

Let's learn about the Android block in the next pill 🚀

[Next: android configuration >]({{ baseurl }}/androidcourse/pill8/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
