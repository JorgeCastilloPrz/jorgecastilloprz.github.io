---
layout: page
current: androidcourse
title: Adding a password input
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Let's use the chance to learn how to add a maximum number of characters for the input 🤔

```xml
<!-- res/layout/activity_main.xml -->

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
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

    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/textField"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:hint="@string/login_input_hint"
        app:counterEnabled="true"
        app:counterMaxLength="@integer/login_input_max_chars">

        <com.google.android.material.textfield.TextInputEditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            />

    </com.google.android.material.textfield.TextInputLayout>
</LinearLayout>
```

We are adding the `app:counterEnabled` and `app:counterMaxLength` properties to enable the behavior and setting the max cap. We are adding the maximum number of characters as an `@integer` resource. Integers can be added to a file in `res/values/integers.xml` like this one:

```xml
<!-- app/src/main/res/values/integers.xml -->
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <integer name="login_input_max_chars">12</integer>
</resources>
```

This gives us a count below the input that grows automatically while we type.

<img src="../../assets/images/input max characters count 1.png" alt="Android Studio" style="width:300px;">

It goes red whenever the maximum is exceeded.

<img src="../../assets/images/input max characters count 2.png" alt="Android Studio" style="width:300px;">

The only issue with this approach is that the user can keep inserting characters even after exceeding the maximum limit, so it is more like a UI hint than anything else. It is possible to combine this feature with the `android:maxLength` attribute for the `TextInputEditText` so the field doesn't allow to type more when you reach the cap 👇

```xml
<com.google.android.material.textfield.TextInputLayout
  android:id="@+id/textField"
  android:layout_width="match_parent"
  android:layout_height="wrap_content"
  android:layout_margin="16dp"
  android:hint="@string/login_input_hint"
  app:counterEnabled="true"
  app:counterMaxLength="@integer/login_input_max_chars">

  <com.google.android.material.textfield.TextInputEditText
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:maxLength="@integer/login_input_max_chars"
    />

</com.google.android.material.textfield.TextInputLayout>
```

<img src="../../assets/images/input max characters count 3.png" alt="Android Studio" style="width:300px;">

Still it would be nice if we implement some validation in our domain to make sure there is no way our app can send a longer username to the server. We will learn how to add logics to our app soon.

If you are curious about the rest of the features for the `TextInputLayout` and `TextInputEditText` combo, you can check [the official guidelines](https://material.io/components/text-fields/android#using-text-fields).

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill16).

[Next: Adding a password input >]({{ baseurl }}/androidcourse/pill17/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
