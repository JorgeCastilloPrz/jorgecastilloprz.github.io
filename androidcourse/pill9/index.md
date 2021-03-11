---
layout: page
current: androidcourse
title: Application dependencies
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

The final bit we need to learn about the Gradle build scripts is how to configure dependencies for our Android app.

### What we mean by "dependencies"

Applications can depend on libraries for different matters. The Kotlin standard library is a good example, required to use the Kotlin language core features.

Other examples of libraries could be the ones that provide UI components like `swiperefreshlayout` (pull to refresh) or `constraintlayout` (another specific UI component). There are also libraries like `appcompat` and `material` for some default themes, or libraries for networking, persistence, serialization, and much more.

Libraries are needed because the Android SDK is not always enough, but mostly because they can solve a problem efficiently once so we don't need to reinvent the wheel or paste the same code on all the applications we write.

One interesting point of libraries is that you can update them independently of the Android SDK version, and therefore pick the right version you want to use per library, hence decide when it's the right time to update.

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

This is where we declare the depdendencies for our `app` module.

Here we see many libraries added as dependencies by default by Android Studio when we created the project. We can go ahead and remove some of them. These are the `dependencies` we want to keep for now:

```groovy
dependencies {
    implementation 'androidx.core:core-ktx:1.3.2'
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.3.0'
}
```

> Note that the Kotlin standard library is added implicitly by the Kotlin Gradle Plugin since some versions ago. We don't need to add it explicitly anymore.

Then **don't forget to click on the "Sync Project with Gradle Files" button** close to the top right corner 🚨

<img src="../../assets/images/gradle sync.png" alt="Android Studio" style="width:400px;">

That will make sure all the required dependencies are fetched and the project is ready to build after the changes. Alternatively, you can click on the "Sync now" text within the banner in the top edge (see on the image also). Both are equivalent.

---

And with this we are finally ready to start adding content to our app in the next block! 🎊

[Next: Adding a screen to our app >]({{ baseurl }}/androidcourse/pill10/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
