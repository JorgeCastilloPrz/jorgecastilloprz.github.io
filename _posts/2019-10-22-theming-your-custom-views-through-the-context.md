---
layout: post
current: post
cover: assets/images/painting6.jpeg
navigation: True
title: Putting custom view theming into Context
date: 2019-10-16 12:36:00
tags: [android, kotlin]
class: post-template
subclass: 'post'
author: jorge
---

Some words about how to theme views programmatically by the Context they're inflated with.

My first recommendation before reading this post would be to take a look to [how to use theme attributes to theme your Android apps](https://jorgecastillo.dev/dependency-inversion-on-android-theming) 🎨 if you didn't yet, since it's very related.

I also highly recommend watching [this talk by Nick Butcher and Chris Banes about the differences between styles and themes](https://www.youtube.com/watch?v=Owkf8DhAOSo). It contrasts styles and themes, and explains super interesting things like how "parent" themes are applied per `Context` and for complete hierarchies inflated using that given `Context`.

Finally, [here's also a nice read](https://ataulm.com/2019/10/14/material-theme-overlay/) by [Ataulm](https://twitter.com/ataulm) about theme overlays that also contains some interesting bits about how the `Context` can be themed.

### Styles and themes per View

Styling `Views` directly on XML layouts is okay, as long as you **reuse a common set of styles across the app to keep coherence**, since that's the ultimate goal, isn't it?

On top of that, you can also apply themes directly on `Views` by using the `android:theme` attribute. That will apply the given theme to that specific `View` and **cascade down to all its descendants**. That's indeed pretty handy to theme certain subsections of your layouts differently.

But both approaches carry an inherent weakness: **They're applied ad hoc per `View` occurrence**. That means they can potentially affect visual coherence across screens if you forget to apply them where required, or by using the wrong style or theme in the wrong place.

### Recommended approach

It is always preferable to have styles or themes **widely imposed by the `ApplicationTheme`**. That leverages coherence and reusability, hence it unlocks design systems (e.g: like Material Design).

You can achieve that by using **default styles**. For deep details on what default styles are, you've got [this talk](https://www.youtube.com/watch?v=Owkf8DhAOSo) and [this post](https://ataulm.com/2019/10/14/material-theme-overlay/), both already recommended above.

For summarizing here just a bit, default styles are standard theme attributes (i.e: `textInputStyle`) that you can give a value to in your `ApplicationTheme`. Those will impose a global style for all the occurrences of the corresponding component across the app (i.e: `TextInputLayout`). Here's a simple example:

```xml
<style name="ApplicationTheme" parent="Theme.MaterialComponents.DayNight">
  <!-- Default style used by material to theme all `TextInputLayouts` equally. -->
  <item name="textInputStyle">@style/ApplicationTheme.Input</item>
</style>

<style name="ApplicationTheme.Input" parent="Widget.MaterialComponents.TextInputLayout.FilledBox">
  <!-- ... -->
</style>
```

If you're able to theme your complete application using default styles defined in your `ApplicationTheme`, and you make sure that your activities are inheriting from this theme, you'll unlock **theming by Context**.

### Theming by Context

Let's say you've got your design system in place and imposed by your `ApplicationTheme`. That means you have got all your activities inheriting the proper themes, and styles and views in the app reusing theme attributes like `?attr/colorSurface` for their colors.

That makes you able to swap your `ApplicationTheme` by any other theme and the app will automatically update its colors transparently.

This happens essentially because your **`ApplicationTheme` is applied to the application `Context` and the `Context` of your activities** under this scenario, hence views inflated using any of those `Contexts` will be themed using the `ApplicationTheme`. Same thing will happen for other elements like `Dialogs`, `DialogFragments`, `BottomSheets`, or any other UI bit.

So here's a remainder that **it's the closest `Context` to the `View` the one imposing how the `View` (or other UI elements) looks**.

As said before, and on top of this, if you want to override this for a specific `View` and descendants, you can use `android:theme` attribute directly on the `View`.

### Code under the hood and implications

For themes, `Context` is wrapped in [`ContextThemeWrapper`](https://developer.android.com/reference/androidx/appcompat/view/ContextThemeWrapper) automatically before being passed to view parent constructors, and the proper application theme is passed for it. Following docs, a `ContextThemeWrapper` is:

> A context wrapper that allows you to modify or replace the theme of the wrapped context.

So, the biggest implication for this fact is that you must take care of using the closest `Context` whenever you're inflating your own views **in code**. Same for dialogs and everything. Otherwise they can potentially skip the theme needed.

But what if you don't have access to that `Context`, or you want to override it from code for any reason? Well you can do what libraries like `material` are already doing, and use `ContextThemeWrapper` by yourself for that.

```kotlin
val themedContext = ContextThemeWrapper(
  context,
  R.style.YourCustomTheme
)

val view = MyCustomView(themedContext)
val otherView = LayoutInflater.from(themedContext).inflate(...)
```

This might not feel like an issue you fall into if you use app wide theming through default styles, but it can come handy in legacy codebases where many components are not following this pattern and are using some random styles. It's usually not easy to refactor the whole thing all at once.

Another scenario where you can find an issue is on instrumentation tests or when you're unit testing Android UI using tools like Robolectric or similar. Whenever you're using a `Context` for instantiating your own views in tests, you should always ensure it's properly themed. Otherwise if you're asserting over UI you can potentially get some assertion errors, since the view will not be themed properly as it is in the production code. Or even worse, if those `Views` are reusing theme attributes (i.e: `?attr/colorSurface`), those attributes will not be resolved and crash at runtime 🙀, so your test will fail.

To solve it, once again, just grab the context and wrap it with a [`ContextThemeWrapper`](https://developer.android.com/reference/androidx/appcompat/view/ContextThemeWrapper) to provide the theme you want for it, before proceeding to inflate any views using it.

You can grab `ContextThemeWrapper` [from the SDK](https://developer.android.com/reference/android/view/ContextThemeWrapper) and it was actually also ported to [AppCompat](https://developer.android.com/reference/androidx/appcompat/view/ContextThemeWrapper) so you can also grab it from there. **Both are equivalent** for what is worth for the end user. The Android team did this mostly for backporting a specific constructor that the Android team needed.

AndroidX Appcompat artifact:

```groovy
implementation "androidx.appcompat:appcompat:1.1.0"
testImplementation "androidx.appcompat:appcompat:1.1.0"
```

### Final words

The difference between themes and styles is highly important, so I can't recommend enough [this talk](https://www.youtube.com/watch?v=Owkf8DhAOSo). Never try to use themes as styles, or vice versa.

If you’re interested in any topics regarding Android, you can [find me on Twitter](https://twitter.com/JorgeCastilloPR).

See you soon 👋
