# Layout, Animation, and Graphics

## Animate bounds with lookahead

`Modifier.animateBounds` animates size and position changes inside a lookahead
scope. Lazy grids and pagers support lookahead and separate lookahead from
approach passes when deciding which items to compose, retain, dispose, scroll,
and target for item animations (1.8.0).

Keyframes with Arcs and Splines and the `AnimatedImageVector` API suite are
stable. In shared-element code, the argument formerly named `state` is
`sharedContentState` (1.8.0).

## Build and migrate shared transitions

Shared-transition APIs are stable and support dynamic enablement, fallback
target bounds for disposed targets, initial gesture velocity, lookahead-scope
coordinates, and `Modifier.skipToLookaheadPosition`. Skip-to-size and
skip-to-position modifiers are active only during a shared transition by
default (1.10.0).

Migrate the finalized names and shapes:

- Replace removed `ScaleToBounds` with `scaleToBounds`.
- Remove the lambda-taking `SharedContentConfig` factory.
- Remove the `clipInOverlayDuringTransition` parameter.
- Treat `BoundsTransform` as following `SizeTransform`.

Use `unveilIn` and `veilOut` as `EnterTransition` and `ExitTransition` options
when an overlay should animate in front of entering or exiting
`AnimatedVisibility` or `AnimatedContent` (1.10.0).

## Debug lookahead animation

`LookaheadAnimationVisualDebugging`,
`CustomizedLookaheadAnimationVisualDebugging`, and
`LookaheadAnimationVisualDebugConfig` visualize target bounds, animation
trajectories, shared-element matching, and active transition state for shared
elements and `Modifier.animatedBounds` (1.11.0).

## Account for animation-state behavior

- `SeekableTransitionState` handles changes made off the UI thread through
  `Snapshot.withMutableSnapshot()` without trying to process the transition on
  that thread (1.11.0).
- `InfiniteRepeatableSpec` prevents zero-duration cycles (1.11.0).
- A custom `AnimationSpec.visibilityThreshold` is honored by
  `animateFloatAsState` (1.11.0).

## Replace deprecated flow layouts

`ContextualFlowRow` and `ContextualFlowColumn` are deprecated. Experimental
`FlowRow` and `FlowColumn` overloads with an `overflow` parameter are also
deprecated. Prefer the overloads without `overflow`, which continue to clip.
Many contextual-row cases can use `FlowRow`; uncommon cases need a custom
layout (1.8.0).

## Use FlexBox for flexible one-dimensional flow

`FlexBox` is a configurable superset of `Row`, `Column`, `FlowRow`, and
`FlowColumn`. `FlexBoxConfig` and `Modifier.flex` control grow, shrink,
wrapping, direction, and alignment. Its DSL uses calls such as `grow(1f)`, not
property assignment. Content that cannot shrink enough overflows the main
axis; add `Modifier.clipToBounds()` when clipping is desired (1.11.0).

## Use Grid for explicit tracks

Experimental `Grid` offers CSS-like explicit two-dimensional layout using
fixed, percentage, flexible, and content-sized tracks plus
`Modifier.gridItem()` placement. Opt in with `ExperimentalGridApi`
(1.11.0).

`GridConfigurationScope.constraints` exposes the available size.
`GridTrackSize.Auto` ranges from min-content to max-content. When a flexible
track contains a `SubcomposeLayout` such as `LazyColumn`, use
`MinMax(0.dp, 1.fr)` to avoid intrinsic queries (1.11.0).

## Build custom lazy layouts

`LazyLayout`, `LazyLayoutItemProvider`, and `LazyLayoutMeasureScope` are stable
with a `LazyLayoutMeasurePolicy`. The empty `LazyLayoutPrefetchState`
constructor and its precomposition and premeasure scheduling methods are also
stable. Custom `PrefetchScheduler` use is deprecated; rely on internal
automatic scheduling (1.9.0).

Foundation adds `BeyondBoundsLayoutModifierNode` for focus-search layout and
`LazyLayoutKeyIndexMap` with a default implementation factory (1.10.0).

## Observe layout and visibility

`Modifier.onLayoutRectChanged` reports root-, window-, or screen-relative
bounds and supports debounce and throttle controls with less overhead than
`onGloballyPositioned`. Read content-container size from
`LocalWindowInfo.current.containerSize`; lint warns against deriving window
size from configuration screen dimensions (1.8.0).

`Modifier.onFirstVisible` and `Modifier.onVisibilityChanged` were introduced
for impression logging, autoplay, and related work (1.9.0). Corrected
`onVisibilityChanged` behavior does not call back for an initially invisible
node and emits `false` after a nonzero `minDurationMs`.
`onVisibilityChangedNode()` exposes the same machinery for a delegatable
custom modifier (1.10.0).

`Modifier.onFirstVisible()` is deprecated because it fires whenever an item
becomes visible again. Use `onVisibilityChanged()` and keep the prior state
needed by the use case (1.11.0).

`Modifier.visible` suppresses drawing without releasing occupied layout space
(1.11.0).

## Implement modifier nodes against focused contracts

`DelegatableNode` receives `onDensityChange` and `onLayoutDirectionChange`.
`PointerInputModifierNode.touchBoundsExpansion` can enlarge a single input
node's hit bounds. `BringIntoViewResponderModifierNode` supplies a node-level,
platform-implementable bring-into-view mechanism (1.8.0).

`UnplacedStateAwareModifierNode` was finalized as
`UnplacedAwareModifierNode`, which is notified when a previously placed layout
becomes unplaced. Rename `DelegatableNode.invalidateLayoutForSubtree` to
`invalidateMeasurementForSubtree` (1.10.0).

A custom modifier node that only needs `onRemeasured()` should implement
`MeasuredSizeAwareModifierNode` instead of the broader
`LayoutAwareModifierNode` (1.11.0).

## Draw shadows, shaders, and layers

Shadow modifier APIs, `DropShadowPainter`, and `InnerShadowPainter` support
custom drop and inner shadows. Share generated shadow infrastructure across
call sites rather than regenerating it each time (1.9.0).

`CompositeShader` and `CompositeShaderBrush` combine two shaders.
`ShaderBrush.transform` applies a transformation matrix. `graphicsLayer`
accepts `blendMode` and `colorFilter` (1.9.0).

`BasicText` no longer creates an implicit `graphicsLayer`; add
`Modifier.graphicsLayer()` explicitly when that layer behavior is needed
(1.8.0).

## Request a per-composable frame rate

Use `Modifier.preferredFrameRate` with a rate or `FrameRateCategory`. It
replaces `requestedFrameRate`, and `FrameRateCategory.NoPreference` was
removed (1.9.0).
