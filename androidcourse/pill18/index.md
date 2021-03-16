---
layout: page
current: androidcourse
title: Adding the login button
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Finally, we want to add the login button so we can submit the form at some point!

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

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="@string/login_button"
        />
</LinearLayout>
```

The button view we've added is simple, as you can see. We are setting `width` and `height`to wrap its content (button text), a margin top so it is not too close to the password input, and a text resource from `res/values/strings.xml`.

```xml
<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_marginTop="16dp"
    android:text="@string/login_button"
    />
```

The text resources:

```xml
<!-- app/src/main/res/values/strings.xml -->
<resources>
    <string name="app_name">AdoptMe</string>
    <string name="login_title">Welcome to AdoptMe 🐶</string>
    <string name="login_input_hint">Please insert your username</string>
    <string name="login_password_hint">Password</string>
    <string name="login_button">Log me in!</string> <!-- 👈 We added this one -->
</resources>
```

<img src="../../assets/images/login button.png" alt="Android Studio" style="width:300px;">

The button is dummy for now, we'll need to trigger some login on click but that will be in the next pill 💊

---

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill18).

[Next: Hooking logic on button click >]({{ baseurl }}/androidcourse/pill19/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
