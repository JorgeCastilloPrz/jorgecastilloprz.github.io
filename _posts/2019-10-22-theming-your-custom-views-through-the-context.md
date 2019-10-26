---
layout: post
current: post
cover: assets/images/painting6.jpeg
navigation: True
title: Putting view theming into Context
date: 2019-10-16 12:36:00
tags: [android, kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Some words about how to theme views in code by the Context they're inflated with.

My first recommendation would be to take a look at [how to use theme attributes to theme your Android apps](https://jorgecastillo.dev/dependency-inversion-on-android-theming) 🎨 since it's very related.

Also, do not miss [this talk by Nick Butcher and Chris Banes from the last Android Developer Summit](https://www.youtube.com/watch?v=Owkf8DhAOSo) about the differences between themes and styles. It contrasts both and highlights things like how "parent" themes are applied per `Context` and for complete `View` hierarchies inflated using it.

Finally, [here's also a nice read](https://ataulm.com/2019/10/14/material-theme-overlay/) by [Ataulm](https://twitter.com/ataulm) about theme overlays that also contains some interesting bits about default styles and how the `Context` is themed.

### Styles and themes per View

Styling `Views` **directly on XML layouts** is okay, as long as you reuse a common set of styles across the app to keep coherence, since that's the ultimate goal, isn't it?

On top of that, you can also theme `Views` by using the `android:theme` attribute directly on them. That applies the theme to that specific `View` and **cascade down to all its descendants**. That comes handy to theme certain subsections of your layouts differently. Keep in mind **this overrides any themes applied at any other level for this `View`**.

Both approaches usually take place when you're not super skilled with styles and themes in Android, so it becomes hard for you to have a proper mental mapping on how to structure those.

Here's their inherent weakness: **They're applied per `View` occurrence**, hence it's tedious and prone to errors. These approaches can potentially affect visual coherence across screens if you forget to apply a style or a theme where required, or by using the wrong ones in the wrong place.

### Recommended approach

It is always preferable to have styles & themes **imposed by the `ApplicationTheme`** whenever possible. This leverages coherence and reusability, plus unlocks design systems (e.g: like Material Design).

You can achieve that by using **default styles**. There's good details on what default styles are on [this talk](https://www.youtube.com/watch?v=Owkf8DhAOSo) and [this post](https://ataulm.com/2019/10/14/material-theme-overlay/), both already recommended above.

For summarizing here a bit, default styles are applied through standard theme attributes (i.e: `textInputStyle`) that you can give a value to in your `ApplicationTheme`. Those will impose a global style for all the occurrences of the corresponding component across the app (i.e: `TextInputLayout`).

```xml
<style name="ApplicationTheme" parent="Theme.MaterialComponents.DayNight">
  <!-- Default style used to theme all `TextInputLayouts`. -->
  <item name="textInputStyle">@style/ApplicationTheme.Input</item>
</style>

<style name="ApplicationTheme.Input" parent="Widget.MaterialComponents.TextInputLayout.FilledBox">
  <!-- Some style item definitions here -->
</style>
```

Then `Views` pass it over to the `super()` constructor as the default style to use. This is how `TextInputLayout` [from Material Components](https://github.com/material-components/material-components-android/blob/e2eec4aca1795c2795f52098e391c85ccc95a1a4/lib/java/com/google/android/material/textfield/TextInputLayout.java#L383) does it.

```kotlin
public TextInputLayout(@NonNull Context context, @Nullable AttributeSet attrs) {
  this(context, attrs, R.attr.textInputStyle);
}
```

Note how **this is a style, not a theme**. The fact that you're using a theme attribute to assign it doesn't mean it's a theme neither a theme overlay. It is a theme `attr` that points to a style, or in other words, **you theme your app using some default styles**.

 The style will inherit from the `material` `TextInputLayout` style as we did in XML above.

If you're able to theme your complete application using default styles defined in your `ApplicationTheme`, and you make sure that your activities are inheriting from this theme, you'll unlock **theming by Context**.

### Theming by Context

So you got your design system in place. Your activities inherit `ApplicationTheme`, and all views in the app should be reusing theme attributes for their colors. Material components already do this by themselves, but you'll want to do the same for any other views.

This makes you able to swap your `ApplicationTheme` by any other theme and the app will automatically update its colors transparently.

**This happens essentially because your app theme is applied to the application Context and the Context of your activities**. 

Any `Views`, `Dialogs`, `DialogFragments`, `BottomSheets`, or any other UI bit in the hierarchy inflated with the same `Context` will be themed following it.

So here's a remainder:

> It's the closest `Context` to the `View` the one imposing how the `View` (or other UI elements) looks.

### Code under the hood and implications

For theming, [material wraps the Context into a ContextThemeWrapper](https://github.com/material-components/material-components-android/blob/4e239315a857189c24e6dbe489115512c1d24762/lib/java/com/google/android/material/theme/overlay/MaterialThemeOverlay.java#L75) before passing it to `View` parent constructors, and the proper application theme is passed for it. This is very well explained by Ataul [on his post](https://ataulm.com/2019/10/14/material-theme-overlay/).

Following docs, a `ContextThemeWrapper` is:

> A context wrapper that allows you to modify or replace the theme of the wrapped context.

The key implication for this fact is that you must take care of using the closest `Context` whenever you're inflating your own views **in code**. Same for dialogs and everything. Otherwise they can potentially skip the theme needed.

Here's how you can use by yourself, in case you don't have some specific `Views` directly linked to the proper `Context` because reasons, or you want to override it from code.

```kotlin
val themedContext = ContextThemeWrapper(
  context,
  R.style.YourCustomTheme // apply the theme you need to the Context.
)

val view = MyCustomView(themedContext)
val otherView = LayoutInflater.from(themedContext).inflate(...)
```

Et voilà, **everything inflated with that `Context` will be themed accordingly**.

This might not feel like an issue you easily fall into if you use app wide theming through default styles, but it can come handy in legacy codebases that are not following this pattern and are using styles arbitrarily. It's usually not easy to refactor the whole thing all at once, is it?

Another scenario where you can find an issue is on instrumentation tests or when you're unit testing Android UI using tools like Robolectric or similar. Whenever you're using a `Context` for instantiating your own views in tests, you should always ensure it's properly themed. Otherwise you will run into two potential failures:

* If those `Views` are reusing theme attributes (i.e: `?attr/colorSurface`), those attributes **will not be resolved and crash** at runtime 🙀, so your test will fail.

* If you're asserting over UI you can potentially get some assertion errors, since the `View` will not be themed as expected.

To solve it, once again, wrap the `Context` with [`ContextThemeWrapper`](https://developer.android.com/reference/androidx/appcompat/view/ContextThemeWrapper) to provide the theme you need before proceeding to inflate any views using it.

You can grab `ContextThemeWrapper` [from the SDK](https://developer.android.com/reference/android/view/ContextThemeWrapper) and it was actually also ported to [AppCompat](https://developer.android.com/reference/androidx/appcompat/view/ContextThemeWrapper) so you can also grab it from there. **Both are equivalent** for what is worth for the end user. The Android team did this mostly for backporting a specific constructor that the Android team needed.

AndroidX Appcompat artifact:

```groovy
implementation "androidx.appcompat:appcompat:1.1.0"
testImplementation "androidx.appcompat:appcompat:1.1.0"
```

### Final words

The difference between themes and styles is highly important, so I can't recommend enough [this talk](https://www.youtube.com/watch?v=Owkf8DhAOSo). Never try to use themes as styles, or vice versa. Writing these posts is also helping me to polish my mental mapping on this.

Don't for get to take a look to [the previous post in the series](https://jorgecastillo.dev/dependency-inversion-on-android-theming), if you didn't!

For anything else you can always [find me on Twitter](https://twitter.com/JorgeCastilloPR), feel free to ping me there.

See you soon! 👋
