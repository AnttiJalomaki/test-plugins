# Layout, Animation, and Graphics

## Lookahead and bounds animation

`Modifier.animateBounds` animates size and position changes inside a lookahead
scope (since 1.8.0). `LazyGrid` and Pager support lookahead, separating the
lookahead and approach passes for scrolling, retained or composed items,
disposal, and item-animation targets.

For visual inspection in 1.11.0, `LookaheadAnimationVisualDebugging`,
`CustomizedLookaheadAnimationVisualDebugging`, and
`LookaheadAnimationVisualDebugConfig` display target bounds, animation
trajectories, shared-element matches, and active-transition state for shared
elements and `Modifier.animatedBounds`.

## Animation API behavior

Keyframes with Arcs and Splines and the `AnimatedImageVector` API suite are
stable in 1.8.0. The `sharedElement` argument previously named `state` is
called `sharedContentState`.

In 1.11.0:

- `SeekableTransitionState` handles off-UI-thread mutations made through
  `Snapshot.withMutableSnapshot()` without processing the transition on that
  thread.
- `InfiniteRepeatableSpec` prevents zero-duration cycles.
- Custom `AnimationSpec` implementations have their `visibilityThreshold`
  honored by `animateFloatAsState`.

## Shared and veil transitions

Shared-transition APIs are stable in 1.10.0. They support dynamic enablement,
fallback target bounds for a disposed target, initial gesture velocity,
lookahead-scope coordinates, and `Modifier.skipToLookaheadPosition`.
Skip-to-size and skip-to-position modifiers are active by default only while a
shared transition is running.

Migration details:

- Replace `ScaleToBounds` with `scaleToBounds`.
- Remove use of the lambda-taking `SharedContentConfig` factory.
- Remove the `clipInOverlayDuringTransition` parameter.
- Define `BoundsTransform` according to the `SizeTransform` contract.

`unveilIn` and `veilOut` are `EnterTransition` and `ExitTransition` options
that animate a veil in front of entering or exiting `AnimatedVisibility` and
`AnimatedContent`.

## Flow layouts and FlexBox

`ContextualFlowRow` and `ContextualFlowColumn` are deprecated in 1.8.0, as are
experimental `FlowRow` and `FlowColumn` overloads with an `overflow`
parameter. Prefer overloads without `overflow`; their behavior remains
`Clip`. Most contextual-row cases can use `FlowRow`; specialized cases may
need a custom layout.

`FlexBox` in 1.11.0 is a configurable superset of `Row`, `Column`, `FlowRow`,
and `FlowColumn`. `FlexBoxConfig` and `Modifier.flex` control growth, shrinkage,
wrapping, direction, and alignment. Its DSL uses function calls such as
`grow(1f)`, not property assignment. Children that cannot shrink enough
overflow the main axis; add `Modifier.clipToBounds()` when that overflow must
be clipped.

## Explicit non-lazy Grid

The experimental `Grid` composable in 1.11.0 provides CSS-like explicit
two-dimensional layout. Opt in with `ExperimentalGridApi`. It supports fixed,
percentage, flexible, and content-sized tracks, while `Modifier.gridItem()`
controls placement.

- `GridConfigurationScope.constraints` exposes the available size.
- `GridTrackSize.Auto` ranges from min-content through max-content.
- Use `MinMax(0.dp, 1.fr)` to avoid intrinsic queries when a flexible track
  contains a `SubcomposeLayout`, such as `LazyColumn`.

## Custom lazy layouts and prefetching

`LazyLayout`, `LazyLayoutItemProvider`, and `LazyLayoutMeasureScope` are stable
in 1.9.0 and pair with `LazyLayoutMeasurePolicy`. The empty
`LazyLayoutPrefetchState` constructor and its precomposition and premeasure
scheduling methods are stable. Custom `PrefetchScheduler` implementations are
deprecated in favor of automatic internal scheduling.

In 1.10.0, Foundation adds `LazyLayoutKeyIndexMap` and a default implementation
factory. `BeyondBoundsLayoutModifierNode` supports layout beyond current
bounds for focus search. The temporary automatic nested-prefetch flag was
removed; delete any assignment.

## Modifier-node hooks and lifecycle

New node hooks in 1.8.0 include:

- `DelegatableNode.onDensityChange` and `onLayoutDirectionChange`.
- `PointerInputModifierNode.touchBoundsExpansion`, which enlarges one pointer
  input node's hit bounds.
- `BringIntoViewResponderModifierNode`, a node-level, platform-implementable
  bring-into-view mechanism.

In 1.10.0, `UnplacedStateAwareModifierNode` is finalized as
`UnplacedAwareModifierNode`, which receives notification when a previously
placed layout becomes unplaced. Rename
`DelegatableNode.invalidateLayoutForSubtree` to
`invalidateMeasurementForSubtree`.

In 1.11.0, a custom node that only needs `onRemeasured()` should implement
`MeasuredSizeAwareModifierNode`, not the broader `LayoutAwareModifierNode`.

## Layout and visibility observation

`Modifier.onLayoutRectChanged` in 1.8.0 observes root-, window-, or
screen-relative bounds. Its debounce and throttle controls make it a lower-
overhead choice than `onGloballyPositioned` for repeated rectangle updates.

`Modifier.visible` in 1.11.0 suppresses drawing while preserving the
composable's occupied layout space. It is not equivalent to removing the
element or measuring it at zero size.

## Shadows, shaders, and layers

Compose 1.9.0 adds customizable shadow modifiers plus `DropShadowPainter` and
`InnerShadowPainter`. Share generated shadow infrastructure across call sites
instead of regenerating it for every use.

`CompositeShader` and `CompositeShaderBrush` combine two shaders, and
`ShaderBrush.transform` applies a shader transformation matrix.
`graphicsLayer` accepts `blendMode` and `colorFilter`.

Compose packed colors cannot be directly compared with Android `ColorLong`
values. Convert explicitly with `toColorLong()` and `fromColorLong()`.

`BasicText` no longer inserts an implicit `graphicsLayer` as of 1.8.0. Add
`Modifier.graphicsLayer()` explicitly only if clipping, isolation, or another
layer behavior is required.

## Per-composable frame-rate requests

Use `Modifier.preferredFrameRate` from `androidx.compose.ui` to request a frame
rate or `FrameRateCategory` (since 1.9.0). It replaces
`requestedFrameRate`; `FrameRateCategory.NoPreference` was removed.
