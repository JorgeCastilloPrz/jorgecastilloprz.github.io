---
layout: page
current: androidcourse
title: Creating our first layout
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Layouts are how we represent static UI in Android. **They are coded with XML**, where **each UI component is a node**. Our first layout will be a blank screen. To do that, we need to add a `resource`.

### Resources

Resources are key in Android. They are like reusable assets for the app. Texts (for the different languages), UI (layouts), images, icons, styles, fonts and much more are added as resources so we can reference them by code and from layouts later. We'll learn all of them by example.

Resources are stored in the `app/src/main/res` directory.

### Adding a layout

To add a `layout` we need to add a `layout` resource folder to the `res` directory.

`right click over res` -> `New` -> `Android Resource Directory`. Then we need to pick `layout` in the dropdown, and forget about the qualifiers box for now.

Once we have the `layout` folder, `right click on it` -> `New` -> `Layout resource file` to add our Layout. Name it `activity_main` and write `LinearLayout` below.

`LinearLayout` is a specific UI component we are using as the root of our layout tree.

> Remember each UI component will be an XML node on the tree.

This is what we can see once the layout is created.

<img src="../../assets/images/our first layout.png" alt="Android Studio" style="width:600px;">

What you see on the right panel is called the "Layout Editor". This is a GUI to drag and drop elements on the screen to make it easier to create UIs. That basically **generates the XML for us**.

The issue is that the XML generated is not as good as it should be many times, and at some point we'll start digging on those XML files to tweak the UI at will anyways.

> Any experienced Android devs out there switched to the XML view at some point on their career and never looked back. That allowed them to unlock the full potential of Android UI without the limitations that a GUI imposes by nature.

In this course I'm skipping the layout editor and going straight to XML. I promise, that will give us a deeper knowledge of how things work for Android UI and we'll become experts on creating neat UIs.

Let's switch to the XML view by selecting the **Split** tab on the top right corner.

<img src="../../assets/images/XML view.png" alt="Android Studio" style="width:400px;">

This is going to give us a double screen split vertically where we can see the XML code on the left and the preview of how that would look on screen on the right so we can have instant feedback loop while we iterate on UI.

This is how our layout XML looks like now:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    >

</LinearLayout>
```

LinearLayout is a UI component that aligns its children vertically or horizontally. `android:orientation="vertical"` is added by default, meaning we could start adding things like texts or buttons inside and those would be aligned vertically.

Its width and height are matching the parent dimensions via the `"match_parent"` attribute. Since this is the root element, the parent is the Activity window that is fullscreen by default. So our `LinearLayout` will also be fullscreen.

If we go back to our activity we should see the `R.layout.activity_main` not colored as red anymore. We've created the file we needed, so we already have our UI logic bound to actual UI 🚀

---

We have our first layout ready, on next pill we are running the application so we can see how it looks on a device 👀

The final version of the code so far can be found [on this branch in GitHub](https://github.com/JorgeCastilloPrz/ultimateandroidcourse/tree/pill11).

[Next: Running our app >]({{ baseurl }}/androidcourse/pill12/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
