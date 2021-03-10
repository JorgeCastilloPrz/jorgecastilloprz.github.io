---
layout: page
current: androidcourse
title: settings.gradle
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

If we open the `settings.gradle` file we will find these lines:

```groovy
include ':app'
rootProject.name = "AdoptMe"
```

As we explained on the previous pill, this file lists all the Gradle modules that will be part of the build. In this case it's only the `app` module. If we had more, all of them would be included the same way like:

```groovy
include ':app', ':domain', ':data'
```

> Local modules are referenced using the colon ":"

The `settings.gradle` can also contain some settings like the project name, as you can see above. The project name matches the root project directory name when it's not explicitly provided.

[Next: build.gradle >]({{ baseurl }}/androidcourse/pill6/)

### Contact me for doubts!

You can find me [on Twitter](https://www.twitter.com/JorgeCastilloPR), where I share all my experiences as a developer, and also [on Instagram](https://www.instagram.com/jorgecastillopr).


Please feel free to contact by any of the mentioned networks or [by mail](mailto:jorge.castillo.prz@gmail.com).
