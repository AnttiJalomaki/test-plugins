# Layout, Animation, and Graphics

## Lookahead and shared transitions

### Bounds animation (1.8.0)

`Modifier.animateBounds` animates size and position changes inside a lookahead
scope. `LazyGrid` and Pager support lookahead, separating lookahead and
approach passes for scrolling, retained and composed items, disposal, and
item-animation targets.

### Shared-transition finalization (1.10.0)

Shared-transition APIs are stable. They add dynamic enablement, fallback
target bounds for disposed targets, initial gesture velocity, lookahead-scope
coordinates, and `Modifier.skipToLookaheadPosition`. Skip-to-size and
skip-to-position modifiers default to active only during a shared transition.

Replace removed `ScaleToBounds` with `scaleToBounds`. Remove the lambda-taking
`SharedContentConfig` factory and `clipInOverlayDuringTransition` parameter.
`BoundsTransform` now follows `SizeTransform`.

### Visual debugging (1.11.0)

`LookaheadAnimationVisualDebugging`,
`CustomizedLookaheadAnimationVisualDebugging`, and
`LookaheadAnimationVisualDebugConfig` visualize target bounds, animation
trajectories, shared-element matches, and active-transition state for shared
elements and `Modifier.animatedBounds`.

## Animation

### Finalized APIs (1.8.0)

Keyframes with arcs and splines and the `AnimatedImageVector` API suite are
stable. Rename the `sharedElement` argument `state` to `sharedContentState`.

### Veil transitions (1.10.0)

`unveilIn` and `veilOut` are `EnterTransition` and `ExitTransition` options
that animate an overlay in front of content entering or exiting
`AnimatedVisibility` and `AnimatedContent`.

### State and spec corrections (1.11.0)

`SeekableTransitionState` handles changes made off the UI thread through
`Snapshot.withMutableSnapshot()` without processing the transition on that
thread. `InfiniteRepeatableSpec` prevents zero-duration cycles. Custom
`AnimationSpec` implementations have their `visibilityThreshold` honored by
`animateFloatAsState`.

## Flow, flex, and grid layouts

### Flow-layout deprecations (1.8.0)

`ContextualFlowRow` and `ContextualFlowColumn` are deprecated, as are
experimental `FlowRow` and `FlowColumn` overloads with an `overflow`
parameter. Prefer overloads without `overflow`; their behavior remains
`Clip`. Most contextual-row cases can use `FlowRow`, while uncommon cases may
need a custom layout.

### FlexBox (1.11.0)

`FlexBox` is a configurable superset of `Row`, `Column`, `FlowRow`, and
`FlowColumn`. Configure grow, shrink, wrapping, direction, and alignment with
`FlexBoxConfig` and `Modifier.flex`.

The DSL uses calls such as `grow(1f)`, not property assignment. Content that
cannot shrink sufficiently overflows the main axis; apply
`Modifier.clipToBounds()` when clipping is required.

### Non-lazy Grid (1.11.0)

Experimental `Grid` provides CSS-like explicit two-dimensional layout with
fixed, percentage, flexible, and content-sized tracks and
`Modifier.gridItem()` placement. Opt in with `ExperimentalGridApi`.

`GridConfigurationScope.constraints` exposes the available size.
`GridTrackSize.Auto` ranges from min-content to max-content. For a flexible
track containing a `SubcomposeLayout` such as `LazyColumn`, use
`MinMax(0.dp, 1.fr)` to avoid intrinsic queries.

## Layout observation and modifier nodes

### Bounds observation (1.8.0)

`Modifier.onLayoutRectChanged` observes root-, window-, or screen-relative
bounds with debounce and throttle controls. It has lower overhead than
`onGloballyPositioned` for this job.

### Node lifecycle hooks (1.8.0)

`DelegatableNode` provides `onDensityChange` and `onLayoutDirectionChange`.
Use these hooks when cached node behavior depends on density or layout
direction.

### Placement and measurement (1.10.0 and 1.11.0)

In 1.10.0, `UnplacedStateAwareModifierNode` is finalized as
`UnplacedAwareModifierNode`, which is notified when a previously placed
layout becomes unplaced. Rename `DelegatableNode.invalidateLayoutForSubtree`
to `invalidateMeasurementForSubtree`.

In 1.11.0, nodes that only need `onRemeasured()` should implement
`MeasuredSizeAwareModifierNode` rather than the broader
`LayoutAwareModifierNode`.

## Drawing, layers, and shaders

### BasicText layers (1.8.0)

`BasicText` no longer inserts an implicit `graphicsLayer`. Add
`Modifier.graphicsLayer()` explicitly when code relies on layer behavior.

### Custom shadows (1.9.0)

Shadow modifiers plus `DropShadowPainter` and `InnerShadowPainter` support
custom drop and inner shadows. Share generated shadow infrastructure across
call sites instead of regenerating it each time.

### Shaders, layers, and packed color (1.9.0)

`CompositeShader` and `CompositeShaderBrush` combine two shaders.
`ShaderBrush.transform` applies a transformation matrix. `graphicsLayer`
accepts `blendMode` and `colorFilter`.

Compose packed colors are not directly comparable to Android `ColorLong`
values. Convert with `toColorLong()` and `fromColorLong()`.

### Frame-rate requests (1.9.0)

Use `Modifier.preferredFrameRate` to request a specific rate or a
`FrameRateCategory` from `androidx.compose.ui`. It replaces
`requestedFrameRate`; `FrameRateCategory.NoPreference` is removed.
