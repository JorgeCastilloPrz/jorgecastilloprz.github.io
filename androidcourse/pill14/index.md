---
layout: page
current: androidcourse
title: Adding a text
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

So far so good! We've got our first app up and running and showing a blank screen with a top bar only 👇

<img src="../../assets/images/our first app.png" alt="Android Studio" style="width:300px;">

This is our `LoginActivity`, so let's add a login box 👩🏾‍💻. The box will be more like an [MVP](https://en.wikipedia.org/wiki/Minimum_viable_product) version of a real login box. It will have a title, an input field, and a button only.

Let's go back to `activity_main.xml` under the `res/layout` folder to add the title first.

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:orientation="vertical"
    >

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textAppearance="@style/TextAppearance.MaterialComponents.Headline6"
        android:text="@string/login_title"
        />
</LinearLayout>
```

We are going to add a few children to the parent `LinearLayout` so they will be aligned vertically given the `android:orientation` attribute.

Since we want all the children to be centered on screen we are adding `android:gravity="center"`. This will make the children center within the parent and at the same time keep their vertical alignment.

Finally, we are adding the `TextView`, which is the Android `View` used to display a text.

> Note that any UI components in Android extend from the View class, hence they're often called "Views". In this example, all components including the parent `LinearLayout` are views.

We have added an `android:id` to the text so we can reference it easily later on. We are also telling to wrap its height and width to the content (actual text) so the view bounds match the text size for both dimensions.

We are also setting the `android:textAppearance` with one of the [Material Design text styles](https://material.io/design/typography/the-type-system.html) to follow the guidelines. We will dig more on those, just note we are picking the "h6" appearance for this one, since it's a header for our login box.

Finally, we are assigning the text via the `android:text` attribute. The text is stored as a string on the `res/values/strings.xml` file:

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">AdoptMe</string>
    <string name="login_title">Insert your username 🙏</string>
</resources>
```

We do this so the text is both reusable and can be localized for the different languages. We will also cover text localization in this course.

We should be able to see the result in real time on the layout preview (right panel) if we have the "Split" option selected 👇

<img src="../../assets/images/our first text.png" alt="Android Studio" style="width:600px;">

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill13).

[Next: Asd >]({{ baseurl }}/androidcourse/pill14/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
