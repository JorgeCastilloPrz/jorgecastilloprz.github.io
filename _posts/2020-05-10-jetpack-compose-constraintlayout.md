---
layout: post
current: post
cover: assets/images/pavo_real.png
navigation: True
title: Jetpack Compose ConstraintLayout
date: 2020-05-01 14:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Learn how to work with ConstraintLayout in Compose.

> Everything listed in this post is part of Jetpack Compose version `0.1.0-dev11`, hence it's quite prone to change over time while the team keeps improving the api surface. ConstraintLayout itself is still under the process of design, so I'll make sure to update this post with further updates.

### 🧠 Concern separation

When `ConstraintLayout` was coded for the Android View system, logics for rendering were decoupled from the `ViewGroup` with the purpose of making it also work for the layout editor. That decision turned out to be pretty convenient so it could also be ported into Jetpack Compose now.

### 🤷🏼‍♀️ How to use it

You can find it in `implementation 'androidx.ui:ui-layout:0.1.0-dev11'` and import it like `import androidx.ui.layout.ConstraintLayout`.

[Here](https://github.com/JorgeCastilloPrz/ComposeConstraintLayoutSamples/) you have a fully working example I prepared and I'll be iterating over in other articles to come.

### 👀 An example

For loading images from a remote URL seamlessly I'll be using Coil via the [Accompanist](https://twitter.com/chrisbanes/status/1261279161735208960) library, by [@chrisbanes](https://twitter.com/chrisbanes). This library provides some utilities for projects in the process of migration to Jetpack Compose, like a theme builder to load theme colors from the closest `Context`, so you can keep using your XML themes and styles while doing the migration.

---

Remember that **you'll need Android Studio 4.1 Canary 8** (Canary 9 does not seem to support Compose) to use Jetpack Compose.

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
