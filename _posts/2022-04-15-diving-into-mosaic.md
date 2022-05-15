---
layout: post
current: post
cover: assets/images/painting8.jpeg
navigation: True
title: Diving into Mosaic for Jetpack Compose
date: 2022-04-26 12:00:00
tags: [compose]
class: post-template
subclass: 'post'
author: jorge
---

Overview on how to create a client library for the Compose compiler and runtime.

## The library

Mosaic is a library created by [@JakeWharton](https://www.twitter.com/JakeWharton) for building console UI that relies on the Jetpack Compose compiler and runtime. Here is a sneak peek on how a counter for the console could be coded (extracted from the library docs):

```kotlin
fun main() = runMosaic {
  var count by mutableStateOf(0)

  setContent {
    Text("The count is: $count")
  }

  for (i in 1..20) {
    delay(250)
    count = i
  }
}
```

`runMosaic` is the integration point, where we can call `setContent` to add some composables to the Composition, or in other words, build and display some UI. These composables can read from Compose snapshot `State` that we can create outside the `setContent` block. This state can be written from the actual program logic, that is also included in the `runMosaic` lambda.

This program updates the `count` state every 250ms, and the UI recomposes accordingly to always reflect the most up to date counter value.

## How does it work?

Mosaic is a nice case study for how to create a **client library for the Jetpack Compose compiler and runtime**. Lets have a look to all its key parts.

#### Nodes

Any Compose client library needs to define its own nodes, and teach the runtime how to materialize them by providing an implementation of the `Applier`. That is: Teaching the runtime how to build and update the tree.

[Here](https://github.com/JakeWharton/mosaic/blob/trunk/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/nodes.kt) are the Mosaic nodes and the Applier implementation.

Normally, in a UI library relying on Jetpack Compose, each UI node knows how to get attached to / detached from the tree, and how to measure, layout, and draw itself. You can see that in [Compose UI LayoutNode](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/node/LayoutNode.k), for example. Mosaic nodes are not different.

As we can see in the link from above, `MosaicNode` is the common node representation used. This class holds information about the node `width` and `height` (calculated when measuring the node), and its `x` and `y` coordinates, relative to the parent. These coordinates are calculated when the parent calls `layout` on the current node.

To measure, layout, and draw itself, `MosaicNode` provides the following abstract methods: `measure()`, `layout()`, and `renderTo(canvas: TextCanvas)`. Those methods are called in order by the `render()` function, also provided by the node. If you are familiar with these phases in Android, the execution order might feel very familiar.

 ```kotlin
internal sealed class MosaicNode {
  //...
  fun render(): String {
    measure()
    layout()
    val canvas = TextSurface(width, height)
    renderTo(canvas)
    return canvas.toString()
  }
  //...
}
```

The node is measured first, then laid out, and finally rendered (drawn).

But `MosaicNode` is only the parent representation of a node. There are two specific implementations available in the library at this point: `TextNode`, for representing text, and `Box`, for representing a container of multiple texts. (Mosaic only displays text).

**TextNode**

It holds its value (actual text), its foreground and background colors, and the text style. `Color` and `TextStyle` are modeled as `@Immutable` classes with a few predefined `@Stable` variants. You can find both types and their variants [here](https://github.com/JakeWharton/mosaic/blob/b44c58bacff0308f26f1a8e32561342d419de7ea/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/components.kt#L32). Flagging these models as `@Immutable` and their variants as `@Stable` is to help the compiler know that these types are reliable for the Compose runtime, so it can trust them to trigger recomposition whenever they change. That said, it is not the purpose of this post to describe class stability in Compose. Such a topic would require its own post. You can learn about it in much detail in the [Jetpack Compose internals book](https://jorgecastillo.dev/book) 📖

Text nodes are measured whenever they get invalidated, which takes place every time the text value is set. To measure, [the Kotlin stdlib `codePointCount`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/code-point-count.html) function is used to calculate the maximum number of Unicode points among all the lines (it could be a multiline text). Code points are numbers that identify the symbols found in text, so they can be represented with bytes. This is a long story, but essentially Mosaic uses the number of code points of the widest line to determine the width of the text. Its height is determined by the number of lines.

Text nodes have a no-op `layout()` function because they don't contain any children to place within them.

For rendering them, it will be delegated into the `TextCanvas` passed to the `renderTo` function, [line by line](https://github.com/JakeWharton/mosaic/blob/b44c58bacff0308f26f1a8e32561342d419de7ea/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/nodes.kt#L56), which will also draw the background, foreground, and text style. If you are curious about how the rendering takes place, you can give a look [to the TextCanvas](https://github.com/JakeWharton/mosaic/blob/trunk/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/canvas.kt).

**BoxNode**

For rendering columns and rows. It keeps a list of its children and a flag to determine if it is used to represent a row or a column. Depending on that, measuring and layout will be done differently.

Measuring the width of a row will measure each children and add up all their widths. The row height will match the height of the tallest children. For measuring columns it is the opposite: The column width will match the child with the biggest width, and column height will be the sum of the height of each child.

The process of laying out children is also simple. If it is a row, it will place them on `y = 0` and `x` will keep growing after each child width, so they are placed one after another, horizontally. If it is a column, all children can be placed at `x = 0` and `y` will grow after each child height, so they are aligned vertically.

Rendering the node means rendering each children, so [it is delegated into the renderTo function of each children](https://github.com/JakeWharton/mosaic/blob/b44c58bacff0308f26f1a8e32561342d419de7ea/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/nodes.kt#L130) (`TextNode`), which ends up delegating to the `TextCanvas`, as explained above. (see [TextCanvas](https://github.com/JakeWharton/mosaic/blob/trunk/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/canvas.kt)).

#### The Applier

We already know the node types used by Mosaic, but the library also needs to teach the runtime how to materialize the node tree. Or in other words, how to build and update the tree. That is done by providing an implementation of the `Applier`. [This is the one used by Mosaic](https://github.com/JakeWharton/mosaic/blob/b44c58bacff0308f26f1a8e32561342d419de7ea/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/nodes.kt#L143):

```kotlin
internal class MosaicNodeApplier(root: BoxNode) : AbstractApplier<MosaicNode>(root) {
  override fun insertTopDown(index: Int, instance: MosaicNode) {
    // Ignored, we insert bottom-up.
  }

  override fun insertBottomUp(index: Int, instance: MosaicNode) {
    val boxNode = current as BoxNode
    boxNode.children.add(index, instance)
  }

  override fun remove(index: Int, count: Int) {
    val boxNode = current as BoxNode
    boxNode.children.remove(index, count)
  }

  override fun move(from: Int, to: Int, count: Int) {
    val boxNode = current as BoxNode
    boxNode.children.move(from, to, count)
  }

  override fun onClear() {
    val boxNode = root as BoxNode
    boxNode.children.clear()
  }
}
```

Appliers decide whether to build the node tree top-down or bottom-up. This decision depends on efficiency, normally. There are examples of the two in Compose UI, which uses one or the other based on the amount of ancestors that need to get notified every time a new node is attached. `LayoutNode`s and `VNode`s (for vectors) build up the tree in two different directions. We are not diving deeper into this here, but you can read [Jetpack Compose internals](https://leanpub.com/composeinternals) if you want to know more.

The most important thing to notice here is how all functions to insert, remove, or move children **delegate those actions into the nodes themselves**. This allows the Compose runtime to stay completely agnostic of implementation details for the specific platform, and let the client library take care of all that. The runtime will simply call the corresponding methods from the applier whenever it needs to.

#### Integration point ⚙️

Any client library also needs to provide an integration point with the platform, where the Composition is created and wired. Mosaic provides [the runMosaic function for this purpose](https://github.com/JakeWharton/mosaic/blob/trunk/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/mosaic.kt).

```kotlin
fun runMosaic(body: suspend MosaicScope.() -> Unit) {
  //...
}

interface MosaicScope : CoroutineScope {
  fun setContent(content: @Composable () -> Unit)
}
```

As we can see, the scope is only a way to scope the `setContent` call in order to hook our composable tree.

Within the `runMosaic` call, we'll see how a root node, a recomposer, and a Composition are created. The recomposer gets a `CoroutineContext` that will be used for all the effects that can run as a result of any recomposition. For this reason, the context is created by adding the current coroutine context (from the `runBlocking` call), a clock for coordination (more on this later), and a job for cancellation (structured concurrency). This is how you are expected to obtain the context when you want to drive recompositions by yourself, like in the case of Mosaic. If you are writing a library that needs to spawn a subcomposition from an existing composition, [rememberCompositionContext()](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#rememberCompositionContext()) can be used.

```kotlin
fun runMosaic(body: suspend MosaicScope.() -> Unit) = runBlocking {
  //...
  var hasFrameWaiters = false
  val clock = BroadcastFrameClock {
    hasFrameWaiters = true
  }

  val job = Job(coroutineContext[Job])
  val composeContext = coroutineContext + clock + job

  val rootNode = BoxNode()
  val recomposer = Recomposer(composeContext)
  val composition = Composition(MosaicNodeApplier(rootNode), recomposer)
  //...
}
```

> Note how the `Composition` gets a new instance of the `Applier`, which has the root node associated. Plus the recomposer.

**Mosaic manages recomposition by itself**, so it coordinates it via its own `MonotonicFrameClock` (see `BroadcastFrameClock` above). For this reason, the next thing we see is a call to `recomposer.runRecomposeAndApplyChanges()`. This is how the recomposition loop is triggered. It listens for invalidations to the compositions registered with it, and then awaits for a frame from the monotonic clock provided with the context when creating the recomposer. The clock will be used to signalize those frames in order to trigger recompositions. The runtime will let Mosaic know when a frame is required via a callback (check the definition of the `BroadcastFrameClock` from above, where it sets `hasFrameWaiters = true` as a result).

```kotlin
fun runMosaic(body: suspend MosaicScope.() -> Unit) = runBlocking {
  //...
  launch(start = UNDISPATCHED, context = composeContext) {
    recomposer.runRecomposeAndApplyChanges()
  }  
  //...
}
```

And here is the logic to signalize frames when the runtime asks for them:

```kotlin
fun runMosaic(body: suspend MosaicScope.() -> Unit) = runBlocking {
  //...
  var displaySignal: CompletableDeferred<Unit>? = null
    launch(context = composeContext) {
      while (true) {
        if (hasFrameWaiters) {
          hasFrameWaiters = false
          clock.sendFrame(0L) // Frame time value is not used by Compose runtime.

          output.display(rootNode.render())
          displaySignal?.complete(Unit)
        }
        delay(50)
      }
  }
  //...
}
```

The loop checks for `hasFrameWaiters` in order to send a new frame and trigger recomposition, every 50ms. `hasFrameWaiters` is set to `true` whenever the clock notifies Mosaic that the runtime requires a frame. On every frame, the complete node tree is rendered and displayed using the output.

Finally, a `CoroutineScope` is created in order to set the `content` to the `Composition` whenever `setContent` is called. This scope is used to run the provided content `body` lambda, passed when calling `runMosaic` at the very beginning. That is where a `setContent` call will be expected.

There is also some dance regarding sending snapshot apply notifications. This is because Mosaic registers a global write observer (see `Snapshot.registerGlobalWriteObserver(observer)` on the snippet) with the goal to allow notifying about changes in state objects that take place outside of a snapshot. When changes take place within a snapshot, the runtime is prepared to notify about those changes automatically, but it does not happen otherwise. This essentially allows Mosaic to support updating mutable state without the requirement to take a `Snapshot` in the client code.

At the end, when the work is complete, the `Job` is cancelled, and the composition disposed, in order to avoid leaks.

```kotlin
fun runMosaic(body: suspend MosaicScope.() -> Unit) = runBlocking {
  //...
  coroutineScope {
    val scope = object : MosaicScope, CoroutineScope by this {
      override fun setContent(content: @Composable () -> Unit) {
        composition.setContent(content)
        hasFrameWaiters = true
      }
    }

    var snapshotNotificationsPending = false
    val observer: (state: Any) -> Unit = {
      if (!snapshotNotificationsPending) {
        snapshotNotificationsPending = true
        launch {
          snapshotNotificationsPending = false
          Snapshot.sendApplyNotifications()
        }
      }
    }
    val snapshotObserverHandle = Snapshot.registerGlobalWriteObserver(observer)
    try {
      scope.body()
    } finally {
      snapshotObserverHandle.dispose()
    }
  }

  //...
  job.cancel()
  composition.dispose()
}
```

#### Creating the nodes 🎨

The last step of the integration is to provide some UI components that create and initialize the nodes. That is required in the case of Mosaic, but also for any other client libraries that display UI. [Here are the ones available in Mosaic](https://github.com/JakeWharton/mosaic/blob/trunk/mosaic/mosaic-runtime/src/main/kotlin/com/jakewharton/mosaic/components.kt) 👇

```kotlin
@Composable
fun Text(
  value: String,
  color: Color? = null,
  background: Color? = null,
  style: TextStyle? = null,
) {
  ComposeNode<TextNode, MosaicNodeApplier>(::TextNode) {
    set(value) {
      this.value = value
    }
    set(color) {
      this.foreground = color
    }
    set(background) {
      this.background = background
    }
    set(style) {
      this.style = style
    }
  }
}

@Composable
fun Row(children: @Composable () -> Unit) {
  Box(true, children)
}

@Composable
fun Column(children: @Composable () -> Unit) {
  Box(false, children)
}

@Composable
private fun Box(isRow: Boolean, children: @Composable () -> Unit) {
  ComposeNode<BoxNode, MosaicNodeApplier>(
    factory = ::BoxNode,
    update = {
      set(isRow) {
        this.isRow = isRow
      }
    },
    content = children,
  )
}
```

`ComposeNode` is provided by the runtime and used to emit a node into the Composition. The node type and Applier type are fixed in the generic type arguments. Then a factory function is provided in order to teach the runtime about how to create the node (a constructor), and the update lambda is used to initialize the node (via the setters).

### More details about the library

For more information about the library and how to use it you can give a read to [the library README](https://github.com/JakeWharton/mosaic).

### Keep learning

If you are interested in the library internals, you can read the [Jetpack Compose internals](https://leanpub.com/composeinternals) book, which includes much deeper content about composition, recomposition, and state.

<img width="300" src="../assets/images/title_page.png"/>

### The original thread

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Mosaic by <a href="https://twitter.com/JakeWharton?ref_src=twsrc%5Etfw">@JakeWharton</a> is a nice case study for how to create a client library for the Jetpack Compose compiler and runtime. Some pointers on where to find the key pieces in Mosaic, if you are interested 🧵<a href="https://t.co/0h3lsn5hSU">https://t.co/0h3lsn5hSU</a> <a href="https://t.co/FklI7G3TGo">pic.twitter.com/FklI7G3TGo</a></p>&mdash; Jorge Castillo (@JorgeCastilloPr) <a href="https://twitter.com/JorgeCastilloPr/status/1525763716318474240?ref_src=twsrc%5Etfw">May 15, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

---

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. See you there!
