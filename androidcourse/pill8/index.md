---
layout: page
current: androidcourse
title: Project structure - android block
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Let me paste the `app/build.gradle` script content here again for context 🙏

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

Time to go over the `android` block!

#### android

This is where we'll fit all the Android related configuration for this module 👇

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

There's a lot to discuss here:

* `compileSdkVersion`: The Android API level Gradle should use to compile our app. This means our app can use the API features included in this API level and lower. `30` stands for Android 11.
* `buildToolsVersion`: The version of the SDK build tools, command-line utilities, and compiler that Gradle should use to build our app. We need to download the build tools using the SDK Manager. Optional since the plugin uses a recommended version of the build tools by default.
* `defaultConfig`: Encapsulates default settings and entries for all build variants. We will learn how to configure flavors to override these values to create different versions of our app.
  * `applicationId`:  Identifier for the application. This ID uniquely identifies your app on the device and in Google Play Store. It's the one we picked as the "package name" when creating the project.
  * `minSdkVersion`: The minimum supported API version. Our app will not be able to get installed in older versions.
  * `targetSdkVersion`: This attribute informs the system that you have tested against the target version and the system should not enable any compatibility behaviors to maintain your app's forward-compatibility with the target version. The application is still able to run on older versions (down to minSdkVersion).
  * `versionCode` and `versionName` describe the version of the app. This one will need to be updated each time we want to publish a new version to the app store.
  * `testInstrumentationRunner` will be used to run Android tests. Not very relevant for now.
* `buildTypes` is used to declare the different build variants we want to create for our app. There is always a default one called "debug", and here we're adding a "release" one.

---

And with this we've got a clear idea on how the build is configured for an Android module ✅ Time to learn about the Android app structure!

[Next: Project Structure - asd >]({{ baseurl }}/androidcourse/pill9/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
