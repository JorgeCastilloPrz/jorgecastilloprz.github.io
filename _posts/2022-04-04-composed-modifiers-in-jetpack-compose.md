---
layout: post
current: post
cover: assets/images/compose-modifiers.jpg
navigation: True
title: Composed modifiers in Jetpack Compose
date: 2022-04-04 10:00:00
tags: [android, compose, modifiers]
class: post-template
subclass: 'post'
author: jorge
---

Learn about composed modifiers and compare those to standard ones.

### Some confusion

Some people seems a bit confused about when to use `composed` to write a custom Modifier in Jetpack Compose, and when to use the `then` extension function instead. Well, they are actually very different. Let me expand on this just a bit.

### "Standard" modifiers

This is an example of a "standard" modifier, using the `then` extension function 👇
```kotlin
@Stable
fun Modifier.padding(all: Dp) =
    this.then(
        PaddingModifier(
            start = all,
            top = all,
            end = all,
            bottom = all,
            rtlAware = true,
            inspectorInfo = debugInspectorInfo {
                name = "padding"
                value = all
            }
        )
    )
```

A "standard" modifier is written using the `Modifier.then` function over the modifier object, or a previous modifier instance. All the modifiers we use daily are written like this, making use of an extension function for ergonomics.

To write a modifier, we must pick the correct type. E.g: `PaddingModifier`, if our aim is to limit the children constraints by introducing some padding, `LayoutModifier` if we want to affect measuring and layout. `ParentDataModifier`, if we need to provide data to a parent during measuring and positioning. `DrawModifier`, if we need to draw into the Layout. There are many others.

```kotlin
private class PaddingModifier(
    val start: Dp = 0.dp,
    val top: Dp = 0.dp,
    val end: Dp = 0.dp,
    val bottom: Dp = 0.dp,
    val rtlAware: Boolean,
    inspectorInfo: InspectorInfo.() -> Unit
) : LayoutModifier, InspectorValueInfo(inspectorInfo) {
    init {
        require(
            (start.value >= 0f || start == Dp.Unspecified) &&
                (top.value >= 0f || top == Dp.Unspecified) &&
                (end.value >= 0f || end == Dp.Unspecified) &&
                (bottom.value >= 0f || bottom == Dp.Unspecified)
        ) {
            "Padding must be non-negative"
        }
    }

    override fun MeasureScope.measure(
        measurable: Measurable,
        constraints: Constraints
    ): MeasureResult {

        val horizontal = start.roundToPx() + end.roundToPx()
        val vertical = top.roundToPx() + bottom.roundToPx()

        val placeable = measurable.measure(constraints.offset(-horizontal, -vertical))

        val width = constraints.constrainWidth(placeable.width + horizontal)
        val height = constraints.constrainHeight(placeable.height + vertical)
        return layout(width, height) {
            if (rtlAware) {
                placeable.placeRelative(start.roundToPx(), top.roundToPx())
            } else {
                placeable.place(start.roundToPx(), top.roundToPx())
            }
        }
    }

    override fun hashCode(): Int {
        var result = start.hashCode()
        result = 31 * result + top.hashCode()
        result = 31 * result + end.hashCode()
        result = 31 * result + bottom.hashCode()
        result = 31 * result + rtlAware.hashCode()
        return result
    }

    override fun equals(other: Any?): Boolean {
        val otherModifier = other as? PaddingModifier ?: return false
        return start == otherModifier.start &&
            top == otherModifier.top &&
            end == otherModifier.end &&
            bottom == otherModifier.bottom &&
            rtlAware == otherModifier.rtlAware
    }
}
```

“Standard" modifiers are **stateless**. Compose UI wraps them to hold their state while resolving / applying them. An example of this is a modifier that affects the size of the node it modifies, or even later modifiers in the chain, like padding. It needs to store the measure somewhere.

```kotlin
Icon(
  painter = rememberVectorPainter(image = Icons.Rounded.Menu),
  contentDescription = "Menu button",
  modifier = Modifier
    .clickable { onMenuClick() }
    .padding(16.dp)
    .background(Color.Red) // only the remaining space after applying the padding will be colored.
)
```

Since the modifier is stateless, all that state can be hold in an internal wrapper that the Compose UI library uses.

### Composed modifiers

But sometimes we actually need to write a stateful modifier. Imagine we need to `remember` things from its body. How could we do it with a stateless one? We need a Composition context. Same for accessing `CompositionLocal`s, for example. That is where you'd use a `composed` modifier. A good example of it is the `clickable` modifier 👇

```kotlin
fun Modifier.clickable(
    interactionSource: MutableInteractionSource,
    indication: Indication?,
    enabled: Boolean = true,
    onClickLabel: String? = null,
    role: Role? = null,
    onClick: () -> Unit
) = composed(
    factory = { ... },
    inspectorInfo = debugInspectorInfo {
        name = "clickable"
        properties["enabled"] = enabled
        properties["onClickLabel"] = onClickLabel
        properties["role"] = role
        properties["onClick"] = onClick
        properties["indication"] = indication
        properties["interactionSource"] = interactionSource
    }
)
```

These `composed` modifiers are **stateful** modifiers that get composed right before getting assigned to the `LayoutNode`, which is the representation of a UI node used by Compose UI. They provide a `@Composable` factory function for creating them in the context of a Composition, so they can call Composable functions like `remember` when created.

```kotlin
fun Modifier.clickable(
    interactionSource: MutableInteractionSource,
    indication: Indication?,
    enabled: Boolean = true,
    onClickLabel: String? = null,
    role: Role? = null,
    onClick: () -> Unit
) = composed(
    factory = {
        val onClickState = rememberUpdatedState(onClick)
        val pressedInteraction = remember { mutableStateOf<PressInteraction.Press?>(null) }
        if (enabled) { ... }
        val isRootInScrollableContainer = isComposeRootInScrollableContainer()
        val isClickableInScrollableContainer = remember { mutableStateOf(true) }
        val delayPressInteraction = rememberUpdatedState { ... }
        val gesture = Modifier.pointerInput(interactionSource, enabled) { ... }
        Modifier
            .then(
                remember { ... }
            )
            .genericClickableWithoutGesture(
                gestureModifiers = gesture,
                interactionSource = interactionSource,
                indication = indication,
                enabled = enabled,
                onClickLabel = onClickLabel,
                role = role,
                onLongClickLabel = null,
                onLongClick = null,
                onClick = onClick
            )
    },
    inspectorInfo = debugInspectorInfo { ... }
)
```

This is one of the topics covered in detail in a brand new chapter about Compose UI available in the [Jetpack Compose Internals](https:leanpub.com/composeinternals) book 🥳

<img width="300" src="../assets/images/title_page.png"/>

If you want to learn much more about the Compose UI library, read [Jetpack Compose Internals](https:leanpub.com/composeinternals) 📖  Here is the full table of contents 👇

**Jetpack Compose Internals: Compose UI**

- Integrating UI with the Compose runtime
- Mapping scheduled changes to actual changes to the tree
- Composition from the point of view of Compose UI
- Subcomposition from the point of view of Compose UI
- Reflecting changes in the UI
- Different types of Appliers
- Materializing a new LayoutNode
- Materializing a change to remove nodes
- Materializing a change to move nodes
- Materializing a change to clear all the nodes
- setContent as the integration point to close the circle
- Measuring in Compose UI
- Measuring policies
- Intrinsic measurements
- Layout Constraints
- Modeling modifier chains
- Setting modifiers to the LayoutNode
- How LayoutNode ingests new modifiers
- Drawing the node tree
- Semantics in Jetpack Compose
- Notifying about semantic changes
- Merged and unmerged semantic trees

Since the book is approaching to a final release, I’ll be raising the price just a bit with every new chapter published, so grab it cheaper while you can! Every purchase will be really appreciated, and boost my motivation to keep writing, even more! 🙏

Thanks for reading! 🙌

---

I also share thoughts and ideas [on Twitter](https://twitter.com/JorgeCastilloPR) quite regularly. See you there!
