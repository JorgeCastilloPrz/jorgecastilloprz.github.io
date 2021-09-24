---
layout: book
current: book
image: assets/images/bookcover.png
title: Jetpack Compose internals 📖
description: A consciously crafted dive into the Jetpack Compose guts.
navigation: true
logo: 'assets/images/ghost.png'
class: page-template
subclass: 'post page'
---

<div class="flex-container-full">
  <div style="flex-grow:1">
    <p style="margin-top:25px;">Do you wonder how Jetpack Compose works internally, or how the compiler or the runtime work together? Are you curious about other use cases for Compose? Did you ever think about how Composable functions communicate with the compiler and the runtime?</p>

    <p>It's your lucky day 🙌 <b>Jetpack Compose internals</b> is your chance to go one step further and learn the guts of what will become the new standard of Android UI.</p>

    <div class="roundedbutton"><a target="_blank" href="https://leanpub.com/composeinternals/">Get the book</a></div>
    <div class="roundedbuttonYellow"><a target="_blank" href="https://twitter.com/composeinternal/">Follow on Twitter</a></div>
  </div>
  <div style="flex-grow:1">
    <img src="../assets/images/title_page.png" alt="My portrait pic" style="width:70%;margin-top:10px">
  </div>
</div>

---

### <span style="font-family: 'Quicksand', 'Noto Sans', sans-serif;">The content</span>


> The book covers following topics among many others.

<div class="flex-container">
  <div>
    <b>1. Composable functions</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. The nature of Composable functions<br/>
    2. Composable function properties<br/>
    3. Calling context<br/>
    4. Idempotent<br/>
    5. Free of side effects<br/>
    6. Restartable<br/>
    7. Fast execution<br/>
    8. Positional memoization<br/>
    9. Similarities with suspend functions<br/>
    10. Composable functions are colored<br/>
    11. Composable function types<br/>
    </p>
  </div>
  <div>
    <b>2. The Compose Compiler</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. A Kotlin compiler plugin<br/>
    2. Compose annotations<br/>
    3. Registering compiler extensions<br/>
    4. Static analysis and static checkers<br/>
    5. Call, type, and declaration checks<br/>
    6. Diagnostic suppression<br/>
    7. Kotlin and runtime version checks<br/>
    8. Code generation (IR) and lowering<br/>
    9. Inferring class stability<br/>
    10. Enabling live literals<br/>
    11. Compose lambda memoization<br/>
    12. Injecting the Composer<br/>
    13. Comparison propagation<br/>
    14. Default parameters<br/>
    15. Control flow group generation<br/>
    16. Klib and decoy generation<br/>
    </p>
  </div>
  <div>
    <b>3. The Compose Runtime</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. The Slot table and the list of changes<br/>
    2. Modeling the changes<br/>
    3. The Composer and how it is fed<br/>
    4. Writing and reading groups<br/>
    5. Remembering values<br/>
    6. Recompose scopes for recomposition<br/>
    7. Side effects in the Composer<br/>
    8. Storing CompositionLocals & source info<br/>
    9. Linking Compositions as a tree<br/>
    10. The current State snapshot<br/>
    11. Navigating the tree nodes<br/>
    12. Performance when building the tree<br/>
    13. Applying the changes<br/>
    14. Attaching and drawing the nodes<br/>
    15. Composition<br/>
    16. Creating a Composition<br/>
    17. The initial Composition process<br/>
    18. Applying changes after Composition<br/>
    19. The Recomposer<br/>
    20. Recomposition process<br/>
    21. Concurrent Recomposition<br/>
    </p>
  </div>  
  <div>
    <b>4. Compose UI</b>
    <p style="font-size:0.8em;margin-top:20px;color:gray;">
    1. To be written<br/>
    2. To be written<br/>
    3. To be written<br/>
    4. To be written<br/>
    5. To be written<br/>
    6. To be written<br/>
    7. To be written<br/>
    8. To be written<br/>
    9. To be written<br/>
    10. To be written<br/>
    11. To be written<br/>
    12. To be written<br/>
    13. To be written<br/>
    14. To be written<br/>
    </p>
  </div>
  <div>
    <b>5. The State snapshot system</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. What snapshost state is<br/>
    2. Concurrency control systems<br/>
    3. Multiversion concurrency control (MVCC)<br/>
    4. The Snapshot<br/>
    5. The snapshot tree<br/>
    6. Snapshots and threading<br/>
    7. Observing reads and writes<br/>
    8. MutableSnapshots<br/>
    9. GlobalSnapshot and nested snapshots<br/>
    10. StateObjects and StateRecords<br/>
    11. Reading and writing state<br/>
    12. Removing or reusing obsolete records<br/>
    13. Change propagation<br/>
    14. Merging write conflicts<br/>
    </p>
  </div>
  <div>
    <b>6. Smart recomposition</b>
    <p style="font-size:0.8em;margin-top:20px;color:gray;">
    1. To be written<br/>
    2. To be written<br/>
    3. To be written<br/>
    4. To be written<br/>
    5. To be written<br/>
    6. To be written<br/>
    7. To be written<br/>
    8. To be written<br/>
    9. To be written<br/>
    10. To be written<br/>
    11. To be written<br/>
    12. To be written<br/>
    13. To be written<br/>
    14. To be written<br/>
    </p>
  </div>  
  <div>
    <b>7. Effects and effect handlers</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. Introducing side effects<br/>
    2. Side effects in Compose<br/>
    3. What we need<br/>
    4. Effect handlers<br/>
    5. Non suspended effects<br/>
    6. Suspended effects<br/>
    7. Third party library adapters<br/>
    </p>
  </div>
  <div>
    <b>8. The Composable lifecycle</b>
    <p style="font-size:0.8em;margin-top:20px;color:gray;">
    1. To be written<br/>
    2. To be written<br/>
    3. To be written<br/>
    4. To be written<br/>
    5. To be written<br/>
    6. To be written<br/>
    7. To be written<br/>
    </p>
  </div>
  <div>
    <b>9. Advanced Compose use cases</b>
    <p style="font-size:0.8em;margin-top:20px;">
    1. Compose runtime vs Compose UI<br/>
    2. Composition of vector graphics<br/>
    3. Building vector image tree<br/>
    4. Integrating vector composition into Compose UI<br/>
    5. Managing DOM with Compose<br/>
    6. Standalone composition in the browser<br/>
    7. Conclusion<br/>
    </p>
  </div>
</div>

### <span style="font-family: 'Quicksand', 'Noto Sans', sans-serif;">The authors and the project</span>

This book is being written by me ([Jorge Castillo](https://www.twitter.com/JorgeCastilloPR)) and [Andrei Shikov](https://twitter.com/shikasd_).

I started writing this book around a year ago, and it helped me to escape a bit from the events happening around and focus on something interesting. It started as a series of posts and suddenly became a book. It is a chance to challenge myself and learn more about a new technology. A great consequence is to be able to share all the gathered knowledge with others 🥳

To write this book we explored the Jetpack Compose sources via [cs.android.com](https://cs.android.com) and sources included in Android Studio. We also had to built our own side projects.

This book aims to gather all the relevant knowledge about Jetpack Compose internals so it works as a reference for the years to come. Jetpack Compose is the future of Android UI, but it will also gain relevance in other platforms given its multiplatform nature.

<div style="height:20px;"></div>

### <span style="font-family: 'Quicksand', 'Noto Sans', sans-serif;">Support me by buying the book 🚀</span>

By buying the book you are not only rewarding the countless ours of effort invested, but also boosting my motivation to keep working on it 🙏

<a href="https://leanpub.com/composeinternals/" target="_blank"><span class="paidtag">BUY THE BOOK 🚀 SINGLE PAYMENT FROM - <span style="color:#000000;">﹩39.99 🤩</span></span></a>

---

<img src="../assets/images/manuelvivo.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Manuel Vivo</b></div>

> "Certainly one of the best resources to learn about the internals of Jetpack Compose. The details in which everything is explained in the book are mind-blowing. 100% recommended for any developer interested in what Jetpack Compose does under the hood. Disclaimer: it does a lot!"

<img src="../assets/images/joeavatar.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Joe Birch</b></div>

> "Being clearly explained and easily consumable, Jorge's Jetpack Compose content has been a great learning resource. I'm really looking forward to picking up this book and soaking up all of the knowledge around these Compose topics!"

<img src="../assets/images/antonioleiva.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Antonio Leiva</b></div>

> "Jorge is able to explain hard concepts in the simplest possible way. This book is another proof of his teaching abilities."

<img src="../assets/images/enrique.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Enrique López-Mañas</b></div>

> "Jorge has done an astonishing job at presenting Compose and its internal to the masses. I can absolutely endorse his book for any developer interested in the peculiarities of Compose and eager to learn more about it."

<img src="../assets/images/andrei.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Andrei Shikov</b></div>

> "Jorge did an amazing job explaining how Compose works in smallest detail, and there's so much more to it than only displaying UI. I thoroughly enjoyed hacking different things based on it, and I hope you will too!"

<img src="../assets/images/portrait.png" alt="My portrait pic" style="width:128px;height:128px">
<div align="center" style="margin-bottom:20px"><b>Jorge Castillo</b></div>

> "This book is one of the hardest challenges I've faced as a developer. I've put all my love and effort on creating a consciously crafted piece of knowledge."

<div style="height:20px;"></div>

### <span style="font-family: 'Quicksand', 'Noto Sans', sans-serif;">Support me by buying the book 🚀</span>

By buying the book you are not only rewarding the countless ours of effort invested, but also boosting my motivation to keep working on it 🙏

<a href="https://leanpub.com/composeinternals/" target="_blank"><span class="paidtag">BUY THE BOOK 🚀 SINGLE PAYMENT FROM - <span style="color:#000000;">﹩39.99 🤩</span></span></a>
