---
layout: post
current: post
cover: assets/images/pavo_real.png
navigation: True
title: Custom Layouts, measuring policies, and BoxWithConstraints in Jetpack Compose
date: 2022-04-03 14:00:00
tags: [android, compose]
class: post-template
subclass: 'post'
author: jorge
---

Measure children according to incoming constraints, and how to defer initial composition for children.

### ↔️ **Layouts and measuring**

In Jetpack Compose any Composable that emits UI is defined as a `Layout`, regardless of what library it belongs to (foundation, material). The `Layout` Composable belongs to Compose UI, which all those libraries depend on for defining their layouts. Think of `Box`, `Column`, `Row`... or any other Composables that emit UI.

When you write a custom `Layout`, you get access to a list of elements to measure (`measurables`), and some Constraints. Those are Constraints imposed by the parent, or a `Modifier` (if there are modifiers affecting the constraints that will be reflected in those Constraints):

```kotlin
@Composable
fun MyBasicColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // measure and position children given constraints logic here
    }
}
```

Inside the block, you can measure the children using the provided constraints, and then call the `layout` function to create the layout and place the children inside. You can read more about this in [the official docs](https://developer.android.com/jetpack/compose/layouts/custom). (The trailing lambda in the `Layout` Composable is the `MeasurePolicy` used to measure the layout and its children).

The way `Layout`s are measured in Jetpack Compose is very similar to how `View`s are measured: It takes place from parent to children (from top to bottom). When a parent needs to measure its children, it imposes some `Constraints` so each child can be measured according to those. Note that by "parent" in this context we can also refer to a `Modifier`, not only a parent `Layout`, since modifiers can also impose additional constraints to a node (e.g: `Modifier.padding`).

### 🤲 A brief example

Let's say we want to render a couple of Composables on screen, where each one of them takes half of the available height.

<img src="/assets/images/withconstraints1.png" alt="WithConstraints sample" width="300px" />

First thought that comes to mind might be measuring screen height then setting half of it to each one of the two Composables. But that would be an ad hoc solution. What would happen if our parent layout didn't use the whole screen height, like the following one?

<img src="/assets/images/withconstraints2.png" alt="WithConstraints sample" width="300px" />

Here, the parent takes only half of the screen height. So what we need is a **responsive** solution that adapts to the available space imposed by the parent.

We could write this using a custom `Layout` in a very performant way:

```kotlin
@Composable
@Preview
fun previewCustomLayout() {
  MyCustomLayout(Modifier.fillMaxSize()) {
    Box(Modifier.background(Color.Magenta))
    Box(Modifier.background(Color.Cyan))
  }
}

@Composable
fun MyCustomLayout(modifier: Modifier, content: @Composable() () -> Unit) {
  Layout(content, modifier) { measurables, constraints ->
    layout(constraints.maxWidth, constraints.maxHeight) {
      val halfHeight = constraints.maxHeight / 2
      val childConstraints = constraints.copy(
        minHeight = minOf(constraints.minHeight, halfHeight),
        maxHeight = halfHeight
      )
      require(measurables.size == 2)
      measurables[0].measure(childConstraints).place(0, 0)
      measurables[1].measure(childConstraints).place(0, halfHeight)
    }
  }
}
```

Here we were able to place the two boxes within the custom layout, since all the information we need to create our Composable tree is already known during the Composition. When the custom layout composes it will also compose its children, and to compose the children we don't need any extra information.

But sometimes we need Composition to actually depend on some value that is not yet available. Imagine that we want to use a different Composable depending on whether we are displaying our UI in a phone or a tablet. We would need to write some conditional logic based on the screen size. But the problem is Composition happens before that information is known. We would need to **defer the composition** for the children somehow. That is where `BoxWithContraints` comes to the rescue.

### Conditional Composition

> The real use case for `BoxWithContraints` is conditional composition.

`BoxWithContraints` is a very special `Layout` that doesn't match the behaviour described above, since it does not compose its children during the composition phase. Children of `BoxWithConstraints` are composed **during the measure/layout stage**, not like the rest of the layouts in compose. This is called **Subcomposition** across the codebase, and it is a bit of a performance overhead (that is why writing custom layouts is recommended first when possible).

> The creator of a subcomposition can control when the initial composition process happens, and `BoxWithConstraints` decides to do it during the layout phase, as opposed to when the root is composed.

For this reason, the real use case for `BoxWithConstraints` is **conditional composition**, like in this example:

```kotlin
BoxWithConstraints {
  if (maxWidth < 560.dp) {
    MyPhoneUi()
  } else {
    MyTabletUi()
  }
}
```

Versioning our layouts depending on the screen dps. A quite familiar scenario in Android isn't it? In this example we want to make a structural change to the Composable tree based on a condition. And it is a condition over a value that we cannot know until the layout phase.

Once we got this, it's a good time to dig into `BoxWithContraints` sources 🎉

### 🕵️‍♀️ Digging into sources

Let's peek into the [BoxWithConstraints sources](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/foundation/foundation-layout/src/commonMain/kotlin/androidx/compose/foundation/layout/BoxWithConstraints.kt;l=59?q=BoxWithConstraints&sq=&ss=androidx) to understand how it works.

```kotlin
/**
* A composable that defines its own content according to the available space,
* based on the incoming constraints or the current [LayoutDirection].
*/
@Composable
fun BoxWithConstraints(
    modifier: Modifier = Modifier,
    contentAlignment: Alignment = Alignment.TopStart,
    propagateMinConstraints: Boolean = false,
    content: @Composable BoxWithConstraintsScope.() -> Unit
) {
    val measurePolicy = rememberBoxMeasurePolicy(
      contentAlignment,
      propagateMinConstraints
    )

    SubcomposeLayout(modifier) { constraints ->
        val scope = BoxWithConstraintsScopeImpl(this, constraints)
        val measurables = subcompose(Unit) { scope.content() }
        with(measurePolicy) { measure(measurables, constraints) }
    }
}
```

This Composable supports children alignment via its `contentAlignment` parameter. That is managed when measuring it. The measuring policy is where measuring logic resides. This is the trailing lambda we pass to any custom `Layout` we write normally. I definitely recommend peeking into the sources of any existing `Layout` in Jetpack Compose, since you can learn a lot about measuring and layout only by reading their measure policies. The measure policy for `BoxWithContraints` starts by double checking if there are no children on it. In that case it measures the box with the minimum possible constraints imposed by the parent:

```kotlin
MeasurePolicy { measurables, constraints ->
  if (measurables.isEmpty()) {
    return@MeasurePolicy layout(
      constraints.minWidth,
      constraints.minHeight
    ) {}
  }
  // ...
}
```

This layout will size itself to fit the content, subject to the incoming constraints. So measuring to the minimum parent constraints when there are no children seems appropriate to match the expectations.

In case there are some children, the next thing done when measuring the `BoxWithContraints` is checking if we want to propagate the minimum parent constraints to those. Most existing layouts either forward down unmodified constraints to their children, or with loose min constraints (set to `0`). Another valid example of this is the `Box` Composable.

```kotlin
MeasurePolicy { measurables, constraints ->
  // ...
  val contentConstraints = if (propagateMinConstraints) {
    constraints
  } else {
    constraints.copy(minWidth = 0, minHeight = 0)
  }
  // ...
}
```

The default behavior is to not propagate them, so minimum constraints will be narrowed down to 0. After getting the constraints ready, it checks if there is only one children, and if that is the case it proceeds to measuring and placing it:

```kotlin
MeasurePolicy { measurables, constraints ->
  // ...
  if (measurables.size == 1) {
    val measurable = measurables[0]
    val boxWidth: Int
    val boxHeight: Int
    val placeable: Placeable
    if (!measurable.matchesParentSize) {
        placeable = measurable.measure(contentConstraints)
        boxWidth = max(constraints.minWidth, placeable.width)
        boxHeight = max(constraints.minHeight, placeable.height)
    } else {
        boxWidth = constraints.minWidth
        boxHeight = constraints.minHeight
        placeable = measurable.measure(
            Constraints.fixed(constraints.minWidth, constraints.minHeight)
        )
    }
    return@MeasurePolicy layout(boxWidth, boxHeight) {
        placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
    }
  }
  // ...
}
```

We find an interesting distinction here. We have two scenarios: sizing the `BoxWithContraints` to wrap its content or sizing it to match its parent.

When the (only) children is not set to match the parent size (e.g: `Modifier.fillMaxSize()`), the `Box` will adapt its size to its wrapped content. To do it, it measures the child first, using the imposed constraints. That returns the size that the child would take if those constraints were imposed. Then the `Box` uses that size to determine its own size. The box `width` will be the maximum value from comparing the `minWidth` from the constraints, and the child width. And the same happens for the height. This effectively means that the box will **never get smaller than its single child**.

In the other hand, when the `Box` is set to match the parent size, it will set its width and height to exactly the same value than the minimum width and height imposed by the parent constraints.

But this only covers the case when there is a single child. What happens when there are more? The policy also defines that, of course. It starts by measuring all the children that are set to **not** match parent size in order to get the size of the `Box`:

```kotlin
MeasurePolicy { measurables, constraints ->
  // ...
  val placeables = arrayOfNulls<Placeable>(measurables.size)

  var hasMatchParentSizeChildren = false
  var boxWidth = constraints.minWidth
  var boxHeight = constraints.minHeight
  measurables.fastForEachIndexed { index, measurable ->
    if (!measurable.matchesParentSize) {
      val placeable = measurable.measure(contentConstraints)
      placeables[index] = placeable
      boxWidth = max(boxWidth, placeable.width)
      boxHeight = max(boxHeight, placeable.height)
    } else {
      hasMatchParentSizeChildren = true
    }
  }
  // ...
}
```

The first line in this snippet initializes a collection of `placeables` to keep track of all of the measured children, since it will need to place them eventually.

After this, it iterates over all the children **that are not set to match the parent size** in order to calculate the maximum width and height possible between min constraints imposed by the parent, and the width and height that each child would take for those constraints. This effectively makes the `Box` adapt to its children the same way than above. Since this process requires measuring each child, it will also use the chance for adding the resulting placeables to the list. Note that any children than are set to **match the parent size** are ignored during this process, since they will be accounted for measuring the `Box` in the next step.

When measuring children that are set to match the parent size, there is something interesting to highlight: If we have unbounded constraints calculated so far for the `BoxWithConstraints`, it will imply setting the minimum constraints (for measuring each child) to 0. That means that each child will decide how narrow it wants to be. This scenario is only possible when the `boxWidth` or `boxHeight` calculated so far are equal to the infinity, which can happen only if the minimum dimensions imposed by the parent in the constraints were unbounded. In case they are not, the already calculated `boxWidth` and `boxHeight` will be used:


```kotlin
if (hasMatchParentSizeChildren) {
  // The infinity check is needed for default intrinsic measurements.
  val matchParentSizeConstraints = Constraints(
    minWidth = if (boxWidth != Constraints.Infinity) boxWidth else 0,
    minHeight = if (boxHeight != Constraints.Infinity) boxHeight else 0,
    maxWidth = boxWidth,
    maxHeight = boxHeight
  )
  measurables.fastForEachIndexed { index, measurable ->
    if (measurable.matchesParentSize) {
      placeables[index] = measurable.measure(matchParentSizeConstraints)
    }
  }
}
```

At the bottom most of the previous snippet we can see how all the children that match parent size are measured using the calculated constraints, and added to the list of `placeables`.

The final action in this measure policy is essentially to create the layout using the calculated width and height, and place all the children inside.

```kotlin
MeasurePolicy { measurables, constraints ->
  // ...
  layout(boxWidth, boxHeight) {
    placeables.forEachIndexed { index, placeable ->
      placeable as Placeable
      val measurable = measurables[index]
      placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
    }
  }
  // ...
}
```

Cheers! we have reached the end of it. We got a proper measuring policy in place for our `BoxWithContraints`. But this only explains how the box and its children are measured, and how the children are placed once measured. Where is the stuff regarding the subcomposition that I mentioned earlier in this post? Well, we only need to step back to the Composable definition to spot it, right at the bottom:

```kotlin
@Composable
fun BoxWithConstraints(
  modifier: Modifier = Modifier,
  contentAlignment: Alignment = Alignment.TopStart,
  propagateMinConstraints: Boolean = false,
  content: @Composable BoxWithConstraintsScope.() -> Unit
) {
  val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
  SubcomposeLayout(modifier) { constraints ->
    val scope = BoxWithConstraintsScopeImpl(this, constraints)
    val measurables = subcompose(Unit) { scope.content() }
    with(measurePolicy) { measure(measurables, constraints) }
  }
}
```

This Composable uses `SubcomposeLayout` instead of `Layout`. `SubcomposeLayout` is an analogue of `Layout` that creates and runs an independent composition (subcomposition) during the layout phase. This allows child Composables to depend on any values calculated in it, like available width for example (since it has been calculated already). To calculate the measurables it simply calls the `subcompose` function to perform subcomposition of the provided content lambda, which will include all its children, and finally proceed to measure them using the calculated measure policy and the incoming parent constraints.

### Learn more about this

This is one of the topics covered in detail in a brand new chapter about Compose UI available in the [Jetpack Compose Internals](https:leanpub.com/composeinternals) book. Give it a read if you want to know more about the different phases (composition, measuring, layout, drawing), measure policies, Subcomposition, and much more.

<img width="300" src="../assets/images/title_page.png"/>

### Where you can find me

I share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. You can also find me [on Instagram](https://www.instagram.com/jorgecastillopr/). See you there!

Stay tunned for more Jetpack Compose posts 👋
