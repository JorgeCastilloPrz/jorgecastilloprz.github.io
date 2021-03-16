---
layout: page
current: androidcourse
title: Adding a text input
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Time to add a text input to our login screen so the end user can enter her username.

```xml
<!-- res/layout/activity_main.xml -->

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
        android:text="@string/login_title"
        android:textAppearance="@style/TextAppearance.MaterialComponents.Headline6"
        />

    <!-- The input 👇 -->
    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/textField"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:hint="@string/login_input_hint">

        <com.google.android.material.textfield.TextInputEditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            />

    </com.google.android.material.textfield.TextInputLayout>
</LinearLayout>
```

We can add the input right below the title `TextView`. The parent `LinearLayout` will keep them aligned vertically. Let's focus on the input declaration for a second since it's a bit special 👇

```xml
<com.google.android.material.textfield.TextInputLayout
    android:id="@+id/textField"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="16dp"
    android:hint="@string/login_input_hint">

    <com.google.android.material.textfield.TextInputEditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        />

</com.google.android.material.textfield.TextInputLayout>
```

The recommended way to add a text input per the [Material Design official guidelines](https://material.io/components/text-fields/android#using-text-fields) is using a combination of a `TextInputLayout` with an inner `TextInputEditText`.

The `TextInputLayout` wrapper is like a decorator. It will add support for things like a hint, a label, an animation between the two, error states and more.

The inner `TextInputEditText` is the actual text input where you type as a user.

We are using `match_parent` to for both to make the field take all available width so it goes edge to edge, but we're also adding a little margin of `16dp` to it so it doesn't reach the edge. Let's run the app ▶️

<img src="../../assets/images/our first input 1.png" alt="Android Studio" style="width:300px;">

We are also adding a hint via the `android:hint` attribute. The text is stored as a string on the `res/values/strings.xml` file:

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">AdoptMe</string>
    <string name="login_title">Welcome to AdoptMe 🐶</string>
    <string name="login_input_hint">Please insert your username</string>
</resources>
```

Here's how it looks when you tap the input so it gains focus. The hint is **automatically animated to become the label** 🤩

<img src="../../assets/images/our first input 2.png" alt="Android Studio" style="width:300px;">

Even if we are using the default one, text fields support different styles in the Material Guidelines. They also support leading and trailing icons, error states and much more. We'll get to that, but if you are curious you can check [the official guidelines](https://material.io/components/text-fields/android#using-text-fields) around how to use it.

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill15).

[Next: Adding input validation >]({{ baseurl }}/androidcourse/pill16/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
