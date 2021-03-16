---
layout: page
current: androidcourse
title: Adding a password input
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Time to add a password input below the username one. This how the complete layout looks after adding it 🙌

```xml
<!-- res/layout/activity_main.xml -->

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
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

    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/textField"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:hint="@string/login_input_hint"
        app:counterEnabled="true"
        app:counterMaxLength="@integer/login_input_max_chars"
        >

        <com.google.android.material.textfield.TextInputEditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:maxLength="@integer/login_input_max_chars"
            />

    </com.google.android.material.textfield.TextInputLayout>

    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/passwordField"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginHorizontal="16dp"
        android:hint="@string/login_password_hint"
        app:endIconMode="password_toggle"
        >

        <com.google.android.material.textfield.TextInputEditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:inputType="textPassword"
            />

    </com.google.android.material.textfield.TextInputLayout>
</LinearLayout>
```

Let's focus on the password field only 👇

```xml
<com.google.android.material.textfield.TextInputLayout
    android:id="@+id/passwordField"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginHorizontal="16dp"
    android:hint="@string/login_password_hint"
    app:endIconMode="password_toggle"
    >

    <com.google.android.material.textfield.TextInputEditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inputType="textPassword"
        />

</com.google.android.material.textfield.TextInputLayout>
```

We make use of the `app:endIconMode` attribute and set it to `"password_toggle"`. That will automatically display an eye trailing icon that we can click to show or hide the password. Then we add the `android:inputType="textPassword"` attribute to the inner `TextInputEditText` so it shows dots instead of the password characters for security reasons:

<img src="../../assets/images/password input 1.png" alt="Android Studio" style="width:300px;">

<img src="../../assets/images/password input 2.png" alt="Android Studio" style="width:300px;">

If you are curious about the rest of the features for the `TextInputLayout` and `TextInputEditText` combo, you can check [the official guidelines](https://material.io/components/text-fields/android#using-text-fields).

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill17).

[Next: Adding the login button >]({{ baseurl }}/androidcourse/pill18/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
