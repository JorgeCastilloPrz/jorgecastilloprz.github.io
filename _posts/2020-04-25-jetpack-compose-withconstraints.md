---
layout: post
current: post
cover: assets/images/pavo_real.png
navigation: True
title: JetPack Compose - Measuring and WithConstraints
date: 2020-05-01 14:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

This composable is used when we need to measure children according to incoming constraints by their parent.

### ↔️ **Layout measuring**

In Jetpack Compose, composables are backed by `LayoutNodes`, which are one of the possible nodes we can find on the composable tree. Each composable has its own `Layout`.

Compose is similar to Android in terms of how layout measuring works. Parent layouts impose constraints to children, from top to bottom. Children can measure according to those constraints imposed by the parent.

For cases where you need to code a custom layout with its own specific measuring logics, you should follow the same approach. With this you will ensure that it renders according to the available space, and never exceeds boundaries imposed by parent.

In fact, that is a very usual need, and that is why [`WithConstraints`](https://developer.android.com/reference/kotlin/androidx/ui/core/package-summary#withconstraints) exists.

### 🤲 A brief example

Let's say we want to render a couple of composables on screen, where each one of them takes half of the available height.

<img src="/assets/images/withconstraints1.png" alt="WithConstraints sample" width="300px" />

First thought that comes to our minds can be measuring screen height then setting half of it to each one of the two composables. But that'd be an ad hoc solution. What if our parent layout didn't take the whole screen, like the following one?

<img src="/assets/images/withconstraints2.png" alt="WithConstraints sample" width="300px" />

Here the parent is taking only half of the screen height. So what we need is a **responsive** solution that adapts to the available space imposed by the parent.

We could use `WithConstraints` composable for this matter. Here is the code:

```kotlin
WithConstraints { constraints, _ ->
    val boxWidth = with(DensityAmbient.current) { constraints.maxWidth.toDp() }
    val boxHeight = with(DensityAmbient.current) { constraints.maxHeight.toDp() / 2 }
    Column {
        Box(
            Modifier.preferredSize(boxWidth, boxHeight),
            backgroundColor = Color.Magenta
        )
        Box(
            Modifier.preferredSize(boxWidth, boxHeight), 
            backgroundColor = Color.Cyan
        )
    }
}
```

This composable forwards constraints imposed by its parent to childs so they can use them to measure themselves. Here, we make use of those to calculate the width and height for a couple of childs, and therefore we can ensure each child will have half of the parent's height.

We leverage the current `DensityAmbient` to get access to utilities regarding screen density, like the `toDp()` function to translate pixels to `dp`.

Also note how we are setting width of our composables as the max width allowed by the parent constraints.

But precisely in this case, you could have solved the problem with a much simpler and more efficient approach by using `Column` and **weights**:

```kotlin
Column(Modifier.fillMaxHeight()) {
    val modifier = Modifier
        .fillMaxWidth()
        .weight(1f)
    Box(modifier, backgroundColor = Color.Magenta)
    Box(modifier, backgroundColor = Color.Cyan)
}
```

You give both items the same weight, and put them into a `Column` to align them vertically, so weights apply vertically in this case. Pretty much what you'd do for a `LinearLayout` in the Android `View` system.

But let's assume you want a bit more control here over your layout. You can always write your own layout that behaves the way you need. The compose team is working hard on making it an easy task.

```kotlin
@Composable
fun test() {
    MyCustomLayout(Modifier.fillMaxSize()) {
        Box(backgroundColor = Color.Magenta)
        Box(backgroundColor = Color.Cyan)
    }
}

@Composable
fun MyCustomLayout(modifier: Modifier, children: @Composable() () -> Unit) {
    Layout(children, modifier) { measurables, constraints, _ ->
        layout(constraints.maxWidth, constraints.maxHeight) {
            val halfHeight = constraints.maxHeight / 2
            val childConstraints = constraints.copy(
                minHeight = minOf(constraints.minHeight, halfHeight),
                maxHeight = halfHeight
            )
            require(measurables.size == 2)
            measurables[0].measure(childConstraints).place(0.ipx, 0.ipx)
            measurables[1].measure(childConstraints).place(0.ipx, halfHeight)
        }
    }
}
```

Creating your custom `Layout` is handy when you have more complex components to encode in your ui that cannot be coded using any of the compose ui available ones.

So then, when are we supposed to make use of `WithContraints` then? 🤔

Good question, let me explain to you something about how compose ui works.

### 🎨 The three compose UI stages

> This was clarified by [Andrey Kulikov](https://twitter.com/and_kulikov), working on Jetpack Compose.
 
**Composition**

This is when all our `@Composable` functions are executed. During this stage the **constraints are still unknown and compose is not measuring layouts at all**. The analog is inflation from the xml: we just created the Views, but they are not measured yet.

**Measure**

This is the next step when we do measure the layouts which were created during the composition.

**Drawing**

Drawing the measured layouts on the canvas.

Now, for a direct mental mapping, and getting back to the examples above. The example using `Columns` and `weights` would be the most efficient one because we are measuring child **during the composition phase**.

The custom layout example highlights how childs **must be measured and positioned during the measure step**. We have basically reimplemented a subset of the features already offered by `Column` for the sake of the example.

So these would be the suggested ways to approach to writing your own composables to match how compose works internally.

---

`WithConstraints` is a very specific case that doesn't match any of the above, since it does not compose its children during the composition phase. Its implementation actually **postposnes composition of children** until there is some additional information available: The parent constraints.

Childs of `WithConstraints` are composed **during the measure step**, not like the rest of the layouts in compose. This is called **subcomposition** across the Jetpack Compose codebase. And it obviously has a bit of performance overhead. This is why the other options are recommended where possible, instead of `WithConstraints`.

Said that, the real use case for `WithConstraints` is **conditional composition**. Here's an example:

```kotlin
WithConstraints { constraints, _ ->
    if (constraints.maxWidth < with(DensityAmbient.current) { 560.dp.toPx() }) {
        MyPhoneUi()
    } else {
        MyTabletUi()
    }
}
```

Versioning our layouts depending on the screen dps. A quite familiar scenario in Android isn't it.

### 🕵️‍♀️ Digging into sources

Let's peek into the [WithConstraints composable sources](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:ui/ui-framework/src/main/java/androidx/ui/core/Layout.kt;l=452?q=WithConstraints&ss=androidx) to understand how it works.

> *Note these are sources for compose `0.1.0-dev10`, so they will likely vary over time while the library gets polished. Still, they can serve us to understand the purpose of this layout, and the importance of imposed constraints.*

```kotlin
/**
 * A composable that defines its own content according to the available space, based on the incoming constraints.
 * ...
 */
@Composable
fun WithConstraints(
    modifier: Modifier = Modifier,
    children: @Composable() (Constraints, LayoutDirection) -> Unit
) {
    val state = remember { WithConstrainsState() }
    state.children = children
    state.context = ContextAmbient.current
    state.recomposer = currentComposer.recomposer
    state.compositionRef = compositionReference()
    // if this code was executed subcomposition must be triggered as well
    state.forceRecompose = true

    LayoutNode(modifier = modifier, ref = state.nodeRef, measureBlocks = state.measureBlocks)

    // if LayoutNode scheduled the remeasuring no further steps are needed - subcomposition
    // will happen later on the measuring stage. otherwise we can assume the LayoutNode
    // already holds the final Constraints and we should subcompose straight away.
    // if owner is null this means we are not yet attached. once attached the remeasuring
    // will be scheduled which would cause subcomposition
    val layoutNode = state.nodeRef.value!!
    if (!layoutNode.needsRemeasure && layoutNode.owner != null) {
        state.subcompose()
    }
}
```

> This layout places children on 0,0 relative to itself (top left corner). Similar to what a `FrameLayout` would do for Views. For specific alignment between the children you'd need to provide your own container to wrap them.

`WithConstraints` accepts `Modifiers` as any other composables, so you can tweak things like spacings, measures and more on top of what the behavior the layout already provides. Learn more about modifiers on [this detailed post by @joebirch](https://joebirch.co/android/exploring-jetpack-compose-modifiers/), who is writting a ton of interesting posts about Jetpack Compose lately.

A key thing to focus on here would be the `WithConstrainsState`. This is how it's being created and used:

```kotlin
val state = remember { WithConstrainsState() }
state.children = children
state.context = ContextAmbient.current
state.recomposer = currentComposer.recomposer
state.compositionRef = compositionReference()
// if this code was executed subcomposition must be triggered as well
state.forceRecompose = true

LayoutNode(modifier = modifier, ref = state.nodeRef, measureBlocks = state.measureBlocks)
```

This class is in charge of calculating **measuring and positioning** for the children. 

Each time this composable needs to get composed (or recomposed) it will read its rendering state from this class. You can see how `LayoutNode`, which is the actual node representing the layout in the tree relies on the node reference and the **measure blocks calculated by the state**.

We will dive into the state implementation in a second.

Another interesting point here is how state is memoized using the `remember` function, making the `calculation` passed to only be evaluated during the composition phase. Any further recompositions will simply return the memoized value.

```kotlin
@Composable
inline fun <T> remember(calculation: () -> T): T =
    currentComposer.cache(true, calculation)
```

> Note that composables in a tree can be composed multiple times at runtime. Composers are the entities in charge of this.

Let's dive into the state class now. It gets a few properties to perform its work. Here is the declaration:

```kotlin
private class WithConstrainsState {
    lateinit var recomposer: Recomposer
    var compositionRef: CompositionReference? = null
    lateinit var context: Context
    val nodeRef = Ref<LayoutNode>()
    var lastConstraints: Constraints? = null
    var children: @Composable() (Constraints, LayoutDirection) -> Unit = { _, _ -> }
    var forceRecompose = false
    
    val measureBlocks = object : LayoutNode.NoIntrinsicsMeasureBlocks(
      error = "Intrinsic measurements are not supported by WithConstraints"
    ) { /* ... */ }
		
    fun subcompose() { /*...*/ }
}
```

Let's have a look to all the properties one by one for a more didactic approach, and in the end we'll have a look at the `subcompose()` function.

**recomposer**

Obtained by `currentComposer.recomposer` from `WithConstraints`. This composable will need to recompose itself each time constraints vary, since children will need to adapt their measure to those. That is handled in the `measureBlocks` property below.

**compositionRef**

Obtained by `compositionReference()` on the call site. It creates a new or returns an already memoized reference for the current point of composition. Per the docs:

> This can be used to run a separate composition in the context of the current one, preserving ambients and propagating invalidations. 

In other words, this reference is a way to refer to the current point in the tree and be able to recompose the portion of the tree starting on that node.

**context**

Simply the Android `Context` we are used to. Used for triggering recomposition from measure logics when required, since that ultimately creates a `UiComposer` under the hood which is part of the platform and ends up yielding `Views` or `ViewGroups` from our composable hierarchy nodes.

**nodeRef**

Reference to get access to the root layout and obtain children from it. This is key since we'll need to measure each child according to the imposed constraints, so we need access to them. 

> Note that the `children` parameter **does not give access to each one of the children**, since that one is simply a function for the DSL that can be called to render the children, but you can't really reference each children from it.

**lastConstraints**

A variable that holds a reference to the last imposed constraints. It'll be updated with a fresh value each time new constraints come in (different to the current ones).

**children**

This is the lambda on the DSL used to render children on the tree, so we can write our composable call like:

```kotlin
WithConstraints {
  // We place our children here.
}
```

**forceRecompose**

A flag to enforce recomposition. It is only set to true the first time, when we are creating the state. Further recompositions will be imposed only when new constraints (different than the current ones) arrive.

**measureBlocks** 

Here is where the actual measuring logics live. It extends `NoIntrinsicsMeasureBlocks` and it's used by our `LayoutNode` in `WithConstraints` to measure children.

You can find `NoIntrinsicMeasureBlocks` [in AOSP](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:ui/ui-platform/src/main/java/androidx/ui/core/ComponentNodes.kt;l=442?q=ComponentNodes.kt&ss=androidx). You'll notice it's a [`MeasureBlocks`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:ui/ui-platform/src/main/java/androidx/ui/core/ComponentNodes.kt;l=389?q=ComponentNodes.kt&ss=androidx) that overrides all methods to calculate intrinsic width and height and throws an error if they are called, so the only method allowed is `measure`.

> For details on what `intrinsicWidth` and `intrinsicHeight` are, I recommend having a look to the [`IntrinsicMeasurable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:ui/ui-core/src/main/java/androidx/ui/core/IntrinsicMeasurable.kt;l=25?q=IntrinsicMeasura&ss=androidx) interface in AOSP.

Let's peek into the `measure` implementation for the `measureBlocks` calculated by this `WithConstraintState`.


```kotlin
override fun measure(
    measureScope: MeasureScope,
    measurables: List<Measurable>,
    constraints: Constraints,
    layoutDirection: LayoutDirection
): MeasureScope.MeasureResult {
    val root = nodeRef.value!!
    if (lastConstraints != constraints || forceRecompose) {
        lastConstraints = constraints
        root.ignoreModelReads { subcompose() }
        // if there were models created and read inside this subcomposition
        // and we are going to modify this models within the same frame
        // the composables which read this model will not be recomposed.
        // to make this possible we should switch to the next frame.
        FrameManager.nextFrame()
    }

    // Measure the obtained children and compute our size.
    val layoutChildren = root.layoutChildren
    var maxWidth: IntPx = constraints.minWidth
    var maxHeight: IntPx = constraints.minHeight
    layoutChildren.forEach {
        it.measure(constraints, layoutDirection)
        maxWidth = max(maxWidth, it.width)
        maxHeight = max(maxHeight, it.height)
    }
    maxWidth = min(maxWidth, constraints.maxWidth)
    maxHeight = min(maxHeight, constraints.maxHeight)

    return measureScope.layout(maxWidth, maxHeight) {
            layoutChildren.forEach { it.place(IntPx.Zero, IntPx.Zero) }
    }
}
```

The first thing we see is how in case new constraints are imposed (different than the current ones) or the `forceRecompose` flag is `true`, `lastConstraints` ref gets updated and a new composition is triggered:

```kotlin
val root = nodeRef.value!!

if (lastConstraints != constraints || forceRecompose) {
    lastConstraints = constraints
    root.ignoreModelReads { subcompose() }
    // if there were models created and read inside this subcomposition
    // and we are going to modify this models within the same frame
    // the composables which read this model will not be recomposed.
    // to make this possible we should switch to the next frame.
    FrameManager.nextFrame()
}
```
In Compose, `@model` reads are automatically observed when you are inside a composition, a measure lambda like this one, or a drawing lambda. `WithConstraints` has a non so common behavior: **triggering synchronous recomposes within the measure block**. To avoid overlapping two model read observation logics, one of those is disabled with `ignoreModelReads {}`, so the ad-hoc recomposition does not trigger those.

Each time `measure` is called we also need children to be measured according to the imposed constraints.

```kotlin
// Measure the obtained children and compute our size (WithConstraints layout)
val layoutChildren = root.layoutChildren
var maxWidth: IntPx = constraints.minWidth
var maxHeight: IntPx = constraints.minHeight

layoutChildren.forEach {
    it.measure(constraints, layoutDirection)
    maxWidth = max(maxWidth, it.width)
    maxHeight = max(maxHeight, it.height)
}

maxWidth = min(maxWidth, constraints.maxWidth)
maxHeight = min(maxHeight, constraints.maxHeight)

return measureScope.layout(maxWidth, maxHeight) {
    layoutChildren.forEach { it.place(IntPx.Zero, IntPx.Zero) }
}
``` 

A reference to the `WithConstraints` children (`List<LayoutNode>`) is obtained from the root node, so we can iterate over them.

Then **minimum** width and height are read from parent constraints and hold into variables.

Afterwards, code iterates over children to measure each one based on the constraints, and compare resulting measures with the ones we have. The goal is to keep the maximum width and height found among all the childs and the minimum values we got from the constraints.

That will give us the minimum required that's able to fit all children (all overlapped and aligned to the relative 0,0) and that it's also greater or equal than the minimum imposed by incoming constraints.

Finally, we want to make sure that the obtained measures are below the maximum imposed by incoming constraints, so we keep the minimum of the two for each dimension.

At the end, we can use the `measureScope` to `layout` our node using the obtained measures, and place all children in `(0,0)`, as explained above.

To end this article we can take a look to the `subcompose` function to see how constraints are always forwarded to children function so you can access it from the outside.

```kotlin
fun subcompose() {
    val node = nodeRef.value!!
    val constraints = lastConstraints!!
    
    subcomposeInto(context, node, recomposer, compositionRef) {
        children(constraints, node.measureScope.layoutDirection)
    }
    forceRecompose = false
}
```

---

You can find the complete implementation of the state [in AOSP](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:ui/ui-framework/src/main/java/androidx/ui/core/Layout.kt;l=481?q=WithConstraints&ss=androidx), so you can also have the big picture.

### Conclusions

JetPack Compose is about composing atomic elements to create complete layout trees. One of the ultimate goals is reusability, so responsibilities for composables must be well defined and bounded, so there's a composable for each UI pattern with the minimum possible overlap between them. Then they are encouraged to be reused everywhere as small pieces that compose well together to create a UI.

In case you need to code your own composables, reusability is the king concept you want to keep in mind. In terms of reusability you can think of them as how you used to think about Android custom views.

Keep in mind that you will need latest [Android Studio Canaries](https://developer.android.com/studio/preview/) to test JetPack compose, since the framework is still under heavy development. That also means all the implementations showcased on this post are still very prone to vary.

### Where you can find me

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there! 

Stay tunned for more JetPack Compose posts 👋
