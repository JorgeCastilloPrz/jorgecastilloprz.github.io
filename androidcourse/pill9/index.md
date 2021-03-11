---
layout: page
current: androidcourse
title: Application dependencies
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

The final bit we need to learn about Gradle is how to configure dependencies for our Android app.

### What we mean by "dependencies"

Applications can depend on libraries for different matters. Ranging from the Kotlin standard library to any of the Android libraries like the ones to implement a pull to refresh, libraries for networking, persistence, or whatever else.

This is because the Android SDK is sometimes not enough. One interesting point of libraries is that you can update them independently of the Android SDK version, and therefore pick the right version you want to use per library, and decide when it's the right time to update.

### dependencies block

Back to the `app/build.gradle` build script, we can find the `dependencies` block 👇

```groovy
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

Here we see many libraries added as dependencies by default by Android Studio when we created the project. Libraries like the Kotlin standard library, some AndroidX libraries for Kotlin extensions (ktx), `appcompat`, `material`, and some testing libraries that we'll remove from the project until we start adding tests. We want to learn step by step.

Go ahead and remove the following dependencies:

* `org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version`: The Kotlin Standard Library is added implicitly by the Kotlin Gradle Plugin since some versions ago. We do not need to declare it explicitly anymore.
* `junit:junit:4.+`, `androidx.test.ext:junit:1.1.2`, `'androidx.test.espresso:espresso-core:3.3.0'`: These are testing dependencies. We'll get to that later in the course and will add the required ones again.

These are the `dependencies` we want to keep for now:

```groovy
dependencies {
    implementation 'androidx.core:core-ktx:1.3.2'
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.3.0'
}
```

Then **don't forget to click on the "Sync Project with Gradle Files" button** close to the top right corner 🚨

<img src="../../assets/images/gradle sync.png" alt="Android Studio" style="width:400px;">

That will make sure all the required dependencies are fetched and the project is ready to build after the changes. Alternatively, you can click on the "Sync now" text within the banner in the top edge (see on the image also). Both are equivalent.

---

And with this we are finally ready to start adding content to our app! 🎊

[Next: Project Structure - Asd >]({{ baseurl }}/androidcourse/pill10/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
