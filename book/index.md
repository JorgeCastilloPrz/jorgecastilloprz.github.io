---
layout: page
current: book
cover: assets/images/bookcover.png
title: Jetpack Compose internals 📖
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

Don't you wonder how Jetpack Compose works internally? How the compiler or the runtime work together? Are you curious about other use cases for Compose? Did you ever think about how Composable functions communicate with the compiler and the runtime?

You are lucky today 🙌 **Jetpack Compose internals** is your chance to go one step further and learn the guts of what will become the new standard of Android UI.

<img src="../assets/images/title_page.png" alt="My portrait pic" style="width:460px;height:598px">

### Be notified when ready 🚀

<!-- Begin Mailchimp Signup Form -->
<link href="//cdn-images.mailchimp.com/embedcode/slim-10_7.css" rel="stylesheet" type="text/css">
<style type="text/css">
	#mc_embed_signup{background:#fff; clear:left; font:14px Helvetica,Arial,sans-serif; }
	/* Add your own Mailchimp form style overrides in your site stylesheet or in this style block.
	   We recommend moving this block and the preceding CSS link to the HEAD of your HTML file. */
</style>
<div id="mc_embed_signup">
<form action="https://dev.us6.list-manage.com/subscribe/post?u=ca4daf32c3c5e272f68ae5eb0&amp;id=b8f302a8c8" method="post" id="mc-embedded-subscribe-form" name="mc-embedded-subscribe-form" class="validate" target="_blank" novalidate>
    <div id="mc_embed_signup_scroll">
	<label for="mce-EMAIL">Jetpack Compose internals</label>
	<input type="email" value="" name="EMAIL" class="email" id="mce-EMAIL" placeholder="email address" required>
    <!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
    <div style="position: absolute; left: -5000px;" aria-hidden="true"><input type="text" name="b_ca4daf32c3c5e272f68ae5eb0_b8f302a8c8" tabindex="-1" value=""></div>
    <div class="clear"><input type="submit" value="Subscribe" name="subscribe" id="mc-embedded-subscribe" class="button"></div>
    </div>
</form>
</div>

<!--End mc_embed_signup-->

### The content

This book is currently under works 🚧 and will be pre-launched very soon. The book will cover the following topics among many others:

* Composable functions in depth. What they represent and their properties.
* Annotations provided by the Compose runtime and their detailed purpose.
* Static analysis performed by the Compose compiler. Frontend compiler checks in place to help while we write code.
* Compiler IR code generation to enable runtime optimizations and features.
* Architecture of the Compose compiler.
* Architecture of the Compose runtime.
* The slot table. The in-memory representation of the Composable call graph. How and when the runtime stores the relevant data from the Composition.
* Composition and Recomposition. How the Composition is created and how it is updated via Recomposition.
* Recomposition scopes.
* How the runtime materializes the node tree via the Applier.
* Compose UI. How the code we write communicates with the compiler and the runtime. Emitting nodes into the Composition. Adding, moving, replacing, and deleting nodes from the tree.
* The State snapshot system.
* Effects and effect handlers. How they are stored in-memory and when they are triggered. Their lifecycle.
* The Composable lifecycle. How it relates to the Android components lifecycle.
* Advanced Compose use cases. Generic node types. Examples of diverse use cases for the Compose compiler and the runtime.
* Multiplatform support.
* Much more 🚀

### The authors and the project

This book is being written by me ([Jorge Castillo](https://www.twitter.com/JorgeCastilloPR)) and [Andrei Shikov](https://twitter.com/shikasd_).

I started this project more than a year ago, when the pandemics started. I did it as a means to escape a bit from what was happening around and focus on something that seemed very interesting at a first glance. It started as a series of posts and suddenly became a book due to its length. This project was also a chance to challenge myself and learn more about a new technology. A great consequence is to be able to share all the gathered knowledge with others 🥳

To write this book we had to dig into the Jetpack Compose sources via [cs.android.com](https://cs.android.com) and the sources included in Android Studio, and explore the guts of the Compose architecture building our own side projects.

This book aims to gather all the relevant knowledge about Jetpack Compose internals so it works as a reference for the years to come. Jetpack Compose is the future of Android UI, but it will also gain relevance in other platforms given its multiplatform nature.

### How to acquire the book

This book is currently under works 🚧 and will be pre-launched very soon. You can register here to get a notification once it is released. Stay tuned!

### Be notified when ready 🚀

<!-- Begin Mailchimp Signup Form -->
<link href="//cdn-images.mailchimp.com/embedcode/slim-10_7.css" rel="stylesheet" type="text/css">
<style type="text/css">
	#mc_embed_signup{background:#fff; clear:left; font:14px Helvetica,Arial,sans-serif; }
	/* Add your own Mailchimp form style overrides in your site stylesheet or in this style block.
	   We recommend moving this block and the preceding CSS link to the HEAD of your HTML file. */
</style>
<div id="mc_embed_signup">
<form action="https://dev.us6.list-manage.com/subscribe/post?u=ca4daf32c3c5e272f68ae5eb0&amp;id=b8f302a8c8" method="post" id="mc-embedded-subscribe-form" name="mc-embedded-subscribe-form" class="validate" target="_blank" novalidate>
    <div id="mc_embed_signup_scroll">
	<label for="mce-EMAIL">Jetpack Compose internals</label>
	<input type="email" value="" name="EMAIL" class="email" id="mce-EMAIL" placeholder="email address" required>
    <!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
    <div style="position: absolute; left: -5000px;" aria-hidden="true"><input type="text" name="b_ca4daf32c3c5e272f68ae5eb0_b8f302a8c8" tabindex="-1" value=""></div>
    <div class="clear"><input type="submit" value="Subscribe" name="subscribe" id="mc-embedded-subscribe" class="button"></div>
    </div>
</form>
</div>

<!--End mc_embed_signup-->

---

<img src="../assets/images/manuelvivo.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Manuel Vivo</b></div>

> "Certainly one of the best resources to learn about the internals of Jetpack Compose. The details in which everything is explained in the book are mind-blowing. 100% recommended for any developer interested in what Jetpack Compose does under the hood. Disclaimer: it does a lot!"

<img src="../assets/images/antonioleiva.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Antonio Leiva</b></div>

> "Jorge is able to explain hard concepts in the simplest possible way. This book is another proof of his teaching abilities."

<img src="../assets/images/enrique.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Enrique López-Mañas</b></div>

> "Jorge has done an astonishing job at presenting Compose and its internal to the masses. I can absolutely endorse his book for any developer interested in the peculiarities of Compose and eager to learn more about it."

<img src="../assets/images/portrait.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Jorge Castillo</b></div>

> "This book is one of the hardest challenges I've faced as a developer. I've put all my love and effort on creating a consciously crafted piece of knowledge."
