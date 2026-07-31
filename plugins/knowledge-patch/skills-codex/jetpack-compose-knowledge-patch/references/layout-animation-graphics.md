# Layout, Animation, and Graphics

## Lookahead and bounds

### Bounds animation (1.8.0)

`Modifier.animateBounds` animates size and position changes inside a lookahead
scope. `LazyGrid` and Pager support lookahead, keeping lookahead and approach
passes distinct for scrolling, retained and composed items, disposal, and item
animation targets.

### Shared-transition finalization (1.10.0)

Shared-transition APIs are stable. They support dynamic enablement, fallback
target bounds for disposed targets, initial gesture velocity, coordinates in a
lookahead scope, and `Modifier.skipToLookaheadPosition`. Skip-to-size and
skip-to-position modifiers are active only during a shared transition by
default.

Replace `ScaleToBounds` with `scaleToBounds`. The lambda-taking
`SharedContentConfig` factory and `clipInOverlayDuringTransition` parameter are
removed. `BoundsTransform` now follows `SizeTransform`.

### Lookahead visual debugging (1.11.0)

`LookaheadAnimationVisualDebugging`,
`CustomizedLookaheadAnimationVisualDebugging`, and
`LookaheadAnimationVisualDebugConfig` visualize target bounds, trajectories,
shared-element matches, and active transitions for shared elements and
`Modifier.animatedBounds`.

## Animation APIs

### Finalized animation APIs (1.8.0)

Keyframes with arcs and splines and the `AnimatedImageVector` API family are
stable. The `sharedElement` argument formerly named `state` is now
`sharedContentState`; named call sites must migrate.

### Veil transitions (1.10.0)

`unveilIn` and `veilOut` are `EnterTransition` and `ExitTransition` options that
animate an overlay in front of content entering or leaving `AnimatedVisibility`
or `AnimatedContent`.

### Animation-state behavior (1.11.0)

`SeekableTransitionState` handles off-UI-thread changes made through
`Snapshot.withMutableSnapshot()` without processing the transition on that
thread. `InfiniteRepeatableSpec` prevents zero-duration cycles. Custom
`AnimationSpec` implementations have their `visibilityThreshold` honored by
`animateFloatAsState`.

## Flow, flex, and grid layouts

### Flow-layout deprecations (1.8.0)

`ContextualFlowRow` and `ContextualFlowColumn` are deprecated. Experimental
`FlowRow` and `FlowColumn` overloads with an `overflow` parameter are also
deprecated. Prefer overloads without `overflow`, which continue to clip.
Ordinary contextual-row cases can often use `FlowRow`; unusual virtualization
or measurement needs may require a custom layout.

### FlexBox (1.11.0)

`FlexBox` is a configurable superset of `Row`, `Column`, `FlowRow`, and
`FlowColumn`, with grow, shrink, wrapping, direction, and alignment through
`FlexBoxConfig` and `Modifier.flex`. Its DSL uses function calls such as
`grow(1f)`, not property assignment. Content that cannot shrink enough
overflows the main axis; add `Modifier.clipToBounds()` when clipping is wanted.

### Non-lazy Grid (1.11.0)

Experimental `Grid` provides explicit two-dimensional layout with fixed,
percentage, flexible, and content-sized tracks and `Modifier.gridItem()`
placement. Opt in with `ExperimentalGridApi`.

`GridConfigurationScope.constraints` exposes available size. `GridTrackSize.Auto`
ranges from min-content to max-content. When a flexible track contains a
`SubcomposeLayout` such as `LazyColumn`, use `MinMax(0.dp, 1.fr)` to avoid
intrinsic queries.

## Modifier-node infrastructure

### Density, direction, pointer bounds, and bring-into-view (1.8.0)

`DelegatableNode` has `onDensityChange` and `onLayoutDirectionChange` hooks.
`PointerInputModifierNode.touchBoundsExpansion` expands one pointer-input
node's hit bounds. `BringIntoViewResponderModifierNode` supplies a node-level,
platform-implementable bring-into-view mechanism.

### Layout and lazy node migrations (1.10.0)

`UnplacedStateAwareModifierNode` is finalized as
`UnplacedAwareModifierNode`, notified when a formerly placed layout becomes
unplaced. Rename `DelegatableNode.invalidateLayoutForSubtree` to
`invalidateMeasurementForSubtree`. Foundation adds
`BeyondBoundsLayoutModifierNode` for focus-search layout and
`LazyLayoutKeyIndexMap` with a default implementation factory.

### Measurement-only nodes (1.11.0)

Custom nodes that only need `onRemeasured()` should implement
`MeasuredSizeAwareModifierNode` instead of the broader
`LayoutAwareModifierNode`.

## Shadows, shaders, layers, and frame rate

### Custom shadows (1.9.0)

Shadow modifiers, `DropShadowPainter`, and `InnerShadowPainter` support custom
drop and inner shadows. Share generated shadow infrastructure across call
sites rather than recreating it for every draw.

### Shader and packed-color interop (1.9.0)

`CompositeShader` and `CompositeShaderBrush` combine shaders.
`ShaderBrush.transform` applies a shader transformation matrix. A
`graphicsLayer` accepts `blendMode` and `colorFilter`.

Compose packed colors are not directly comparable with Android `ColorLong`
values. Convert using `toColorLong()` and `fromColorLong()`.

### Per-composable frame-rate requests (1.9.0)

Use `Modifier.preferredFrameRate` with a rate or `FrameRateCategory` from
`androidx.compose.ui`. It replaces `requestedFrameRate`.
`FrameRateCategory.NoPreference` was removed.

## Layout checklist

- Distinguish lookahead target measurement from approach-pass animation.
- Do not use a non-lazy `Grid` when content needs lazy composition.
- Clip FlexBox overflow explicitly only when that visual policy is desired.
- Update named arguments after animation API parameter renames.
- Keep node interfaces as narrow as the callbacks actually required.
- Convert platform and Compose packed-color representations before comparison.
