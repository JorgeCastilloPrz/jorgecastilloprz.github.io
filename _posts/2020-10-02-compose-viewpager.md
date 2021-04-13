---
layout: post
current: post
cover: assets/images/painting7.png
navigation: True
title: Jetpack Compose ViewPager
date: 2020-10-02 11:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Let's use a few minutes to learn how to write a swipeable pager composable.

## Disclaimer

In case you got here looking for a Jetpack Compose implementation of a `Pager`, let me suggest you to take a look at [the semi-official one provided by Accompanist](https://google.github.io/accompanist/pager/) by [Chris Banes](https://twitter.com/chrisbanes).

Accompanist provides variants for both `HorizontalPager` and `VerticalPager` and for now it's flagged as an `@ExperimentalPagerApi`, since its api surface might still evolve. That said, I've tested it and this implementation works much better than the one I was diving into in this post.

For that reason I've decided to deprecate the post given the information on it was starting to become misleading and quite outdated given how tied it was to the component implementation details. It already served it's purpose!

I am keeping the post entry since SEO is good enough here so it can help leading devs to the Accompanist implementation for now.

Finally, let me recommend the [official Jetpack Compose samples](https://github.com/android/compose-samples/) as a really good way to get familiarized with Compose codebases and architecture 🚀

---

You might be interested in other posts I wrote about Jetpack Compose:

* [Awaiting next animation frame using suspend](https://jorgecastillo.dev/jetpack-compose-await-next-frame)
* [Sneak peek into Compose ConstraintLayout](https://jorgecastillo.dev/jetpack-compose-constraintlayout)
* [Custom layouts, measuring and WithConstraints Composable](https://jorgecastillo.dev/jetpack-compose-withconstraints)

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for Jetpack Compose posts 🙌
