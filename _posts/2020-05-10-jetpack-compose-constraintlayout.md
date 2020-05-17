---
layout: post
current: post
cover: assets/images/yak.png
navigation: True
title: Jetpack Compose ConstraintLayout
date: 2020-05-01 14:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Learn how to work with ConstraintLayout in Compose.

> Everything listed in this post is part of Jetpack Compose version `0.1.0-dev11`, hence it's quite prone to change over time while the team keeps improving the api surface. I'll make sure to update this post frequently.

### 🧠 Concern separation

When `ConstraintLayout` was coded for the Android View system, logics for rendering (constraints) were decoupled from the `ViewGroup` with the purpose of making it also work for the layout editor. That decision turned out to be pretty convenient so it could also be ported into Jetpack Compose now.

### 🤷🏼‍♀️ How to use it

You can find it in `implementation 'androidx.ui:ui-layout:0.1.0-dev11'` and import it like `import androidx.ui.layout.ConstraintLayout`.

[Here](https://github.com/JorgeCastilloPrz/ComposeConstraintLayoutSamples/) you have a fully working example I prepared and I'll be iterating over in other articles to come.

### 👀 An example

There is [an awesome sample project](https://github.com/riggaroo/ConstraintLayoutDemo) on `ConstraintLayout` and `MotionLayout` by [@riggaroo](https://twitter.com/riggaroo) that showcases how to build a Google Play Movies detail like layout for the Android View System. In this post I'll do an early try to replicate it for Jetpack Compose as possible. Here is the final result:

<table>
	<tr>
	  <th><img style="display: table;" src="/assets/images/aladdin.png" alt="Aladdin sample" width="300px" />
	  </th>
	  <th><img style="display: table;" src="/assets/images/aladdin.gif" alt="Aladdin gif" width="300px" />
	  </th>
	</tr>
</table>

For loading images from a remote URL seamlessly I'll be using Coil via the [Accompanist](https://twitter.com/chrisbanes/status/1261279161735208960) library, by [@chrisbanes](https://twitter.com/chrisbanes). This library provides some utilities for projects in the process of migration to Jetpack Compose, like a theme builder to load theme colors from the closest `Context`, so you can keep using your XML themes and styles while doing the migration.

First, I'm creating the rounded circles with an icon (or a text) inside as a seaprate component so we can reuse it. For those, we'll create a `RoundedIconButton` composable that gets a series of arguments: The `tag`, that will be used to reference composables within our `ConstraintLayout` declaration and we'll look into it later, the image vector asset, a text to put below it and an optional background color.

```kotlin
@Composable
fun RoundedIconButton(
    tag: String,
    asset: VectorAsset,
    text: String,
    background: Color = lightThemeColors.primary
) {
    Column(
        modifier = Modifier.tag(tag) + Modifier.padding(16.dp),
        horizontalGravity = Alignment.CenterHorizontally
    ) {
        Image(
            modifier = Modifier.drawBackground(
                background,
                CircleShape
            ) + Modifier.padding(20.dp),
            asset = asset
        )
        Text(
            modifier = Modifier.padding(top = 8.dp),
            style = MaterialTheme.typography.body2,
            color = Color.Gray,
            text = text
        )
    }
}
```

By adding a `Column` I align the image and the text below it vertically, Then I impose center horizontal aligment for both things inside the `Column` with a modifier.

From the outside, we'll be able to call it like:

```kotlin
RoundedIconButton(
    tag = "familyCircle",
    asset = vectorResource(R.drawable.ic_family),
    text = "Family"
)
```

This is the visual result 👍

<img src="/assets/images/RoundedIconButton.png" alt="Aladdin gif" width="200px"/>

Let's move on now into the actual `ConstraintLayout` code.

I will provide a new component representing my screen for this. Content will be into a `VerticalScroller` since I'll want it to be scrollable in the vertical direction:

```kotlin
@Composable
fun GooglePlayScreen(movie: MovieViewState) {
  VerticalScroller {
    // My screen composable tree will live in here
  }
```

Time to to add our `ConstraintLayout` to the mix.

```kotlin
@Composable
fun GooglePlayScreen(movie: MovieViewState) {
    VerticalScroller {
	ConstraintLayout(
	  modifier = Modifier.drawBackground(color = Color.White) +
	  	Modifier.fillMaxWidth() +
	  	Modifier.fillMaxHeight(),
	  constraintSet = ConstraintSet {
	    // We'll add our initial constraints here!
	  }) {
	    // Our composables will be the composable children declared here.
	    // They will be constrained by the constraints provided above.
	  }
}
```

Since Jetpack Compose composables don't rely on ids for reference like in the `View` system, we need to create `tags` to refer to those within our constraints.

Let's create a tag for our header image, and add some constraints to it:

```kotlin
@Composable
fun GooglePlayScreen(movie: MovieViewState) {
    VerticalScroller {
	ConstraintLayout(
	  modifier = Modifier.drawBackground(color = Color.White) +
	  	Modifier.fillMaxWidth() +
	  	Modifier.fillMaxHeight(),
	  constraintSet = ConstraintSet {
	    val headerImage = tag("headerImage")
	    
	    headerImage.top constrainTo parent.top
	    headerImage.left constrainTo parent.left
	    headerImage.right constrainTo parent.right
	  }) {
	    CoilImage(
                data = movie.headerImageUrl,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxWidth() + 
                  Modifier.preferredHeight(240.dp) + 
                  Modifier.tag("headerImage")
            )
	  }
}
```

We just need add the composable as a children of our `ConstraintLayout`, like I'm doing with my `CoilImage`. Remember to 
**tag the view with the the same tag used for its constraints** ⚠️, using a tag modifier.

You can see how simple is it to constrain our views:

```kotlin
headerImage.top constrainTo parent.top
headerImage.left constrainTo parent.left
headerImage.right constrainTo parent.right
```

Let's add our movie cover image as a portrait:

```kotlin
ConstraintLayout(
  modifier = Modifier.drawBackground(color = Color.White) +
    Modifier.fillMaxWidth() +
    Modifier.fillMaxHeight(),
  constraintSet = ConstraintSet {
    val headerImage = tag("headerImage")
    val portraitImage = tag("portraitImage")
    
    headerImage.top constrainTo parent.top
    headerImage.left constrainTo parent.left
    headerImage.right constrainTo parent.right
	 
    portraitImage.left constrainTo parent.left
    portraitImage.top constrainTo headerImage.bottom
    portraitImage.bottom constrainTo headerImage.bottom
}) {
  CoilImage(
     data = movie.headerImageUrl,
     contentScale = ContentScale.Crop,
     modifier = Modifier.fillMaxWidth() + 
     	  		Modifier.preferredHeight(240.dp) + 
     	  		Modifier.tag("headerImage"))
            
  CoilImage(
    data = movie.portraitUrl,
    modifier = Modifier.preferredSize(120.dp, 260.dp) +
     		Modifier.padding(8.dp) + 
     		Modifier.tag("portraitImage"))
   }
```

We align its top and bottom to the bottom edge of the header image, so that way we center it vertically to that line.

Let's add a title now:

```kotlin
ConstraintLayout(
  modifier = Modifier.drawBackground(color = Color.White) +
    Modifier.fillMaxWidth() +
    Modifier.fillMaxHeight(),
  constraintSet = ConstraintSet {
    val headerImage = tag("headerImage")
    val portraitImage = tag("portraitImage")
    val title = tag("title")
                
    headerImage.top constrainTo parent.top
    headerImage.left constrainTo parent.left
    headerImage.right constrainTo parent.right
	 
    portraitImage.left constrainTo parent.left
    portraitImage.top constrainTo headerImage.bottom
    portraitImage.bottom constrainTo headerImage.bottom
    
    title.top constrainTo headerImage.bottom
    title.left constrainTo portraitImage.right
    title.right constrainTo parent.right
    title.width = spread
}) {
  CoilImage(
     data = movie.headerImageUrl,
     contentScale = ContentScale.Crop,
     modifier = Modifier.fillMaxWidth() + 
     	  		Modifier.preferredHeight(240.dp) + 
     	  		Modifier.tag("headerImage"))
            
  CoilImage(
    data = movie.portraitUrl,
    modifier = Modifier.preferredSize(120.dp, 260.dp) +
     		Modifier.padding(8.dp) + 
     		Modifier.tag("portraitImage"))
   }
   
   Text(
     movie.name,
     style = MaterialTheme.typography.h4,
     modifier = Modifier.padding(top = 16.dp) + Modifier.tag("title"))
```

Note how we align the title to be like:

```kotlin
title.top constrainTo headerImage.bottom
title.left constrainTo portraitImage.right
title.right constrainTo parent.right
title.width = spread
```

That means it'll be aligned below the header image, and to the right of the portrait image. The `spread` width has the same effect than for the `View` system `ConstraintLayout`: Since our text is smaller than the available width space there, we want to take all available space so its content (the actual text within the composable) gets aligned to the left given its gravity. Here's the visual result with the text background in Red so you can see what I'm talking about:

<table>
	<tr>
	  <th><img style="display: table;" src="/assets/images/constraint_header1.png" alt="Aladdin sample" width="300px" />
	  </th>
	  <th><img style="display: table;" src="/assets/images/constraint_header2.png" alt="Aladdin gif" width="300px" />
	  </th>
	</tr>
</table>

And by following this pattern you can add the rest of the views. There are not any gotchas in there, so I'll prompt you to have a look to [the complete screen code here](https://github.com/JorgeCastilloPrz/ComposeConstraintLayoutSamples/blob/ac606f558754586f25c600f81b10b247626890b2/app/src/main/java/dev/jorgecastillo/composeconstraintlayout/movies/GooglePlayScreen.kt#L33).

I plan on iterating a bit over this sample to add some interesting animations to the screen in future posts, so stay tunned 👍

Also keep in mind `ConstraintLayout` is not being promoted yet by the Jetpack Compose team, precisely because it's prone to change, as many other apis. Keep that in mind when you use it 🙏

---

Remember that **you'll need Android Studio 4.1 Canary 8** (Canary 9 does not seem to support Compose) to use Jetpack Compose.

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
