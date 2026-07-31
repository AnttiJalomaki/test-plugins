# Input, Scrolling, and Focus

## Focus

### Navigation and restoration (1.8.0)

Stable focus APIs replace `FocusProperties.enter` and `exit` with
receiver-based `onEnter` and `onExit`. `FocusRequester` and
`FocusTargetModifierNode` add `requestFocus(FocusDirection)`.

`Modifier.focusRestorer()` now accepts a non-null `FocusRequester` parameter
named `fallback`, defaulting to `FocusRequester.Default`; do not pass the old
lambda.

### Indirect-pointer focus behavior (1.10.0)

All indirect-touch APIs are renamed to indirect-pointer APIs. Mouse and
touchpad presses outside the focused node now clear focus by default. Set
`AbstractComposeView.isClearFocusOnPointerDownEnabled = false` to opt out.

Calling `FocusTargetModifierNode.requestFocus()` on a non-focusable node now
routes focus to a child. Use
`DelegatableNode.requestFocusForChildInRootBounds()` to target an overlapping
child. Enable non-touch initial focus with
`ComposeUiFlags.isInitialFocusOnFocusableAvailable`.

## Haptics and gestures

### IndicationNodeFactory migration (1.9.0)

After recompilation, indication-less overloads of `clickable`,
`combinedClickable`, `selectable`, `toggleable`, and `triStateToggleable`
accept only an `IndicationNodeFactory` supplied by `LocalIndication`. Supplying
a deprecated `Indication` can crash at runtime.

Migrate the indication or use an explicit-indication overload as a
compatibility bridge. On 1.9, the temporary last resort is
`ComposeFoundationFlags.isNonComposedClickableEnabled = false`; the flag is
removed in 1.10.0.

### Expanded haptic feedback (1.8.0)

`LocalHapticFeedback` supplies a default Android implementation when the
vibrator reports support. `HapticFeedbackType` values now include `Confirm`,
`ContextClick`, `GestureEnd`, `GestureThresholdActivate`, `Reject`,
`SegmentFrequentTick`, `SegmentTick`, `ToggleOn`, `ToggleOff`, and
`VirtualKey`.

### Gesture dispatch and nested hand-off (1.8.0)

Tap gesture detectors use immediate coroutine dispatch by default. The
compatibility flag is
`ComposeFoundationFlags.isDetectTapGesturesImmediateCoroutineDispatchEnabled`
at this release; it is removed in 1.11.0.

A parent draggable or scrollable can pick up a gesture abandoned by its
child. If a fling reaches a bound, its remaining velocity passes to the next
scrollable in the chain.

### Drag controls and fling threshold (1.9.0)

A `detectDragGestures` overload controls touch slop and orientation locking.
Read `ViewConfiguration.minimumFlingVelocity` for the lower fling threshold.

### Delayed presses (1.11.0)

`ComposeFoundationFlags.isDelayPressesUsingGestureConsumptionEnabled` makes
drag containers delay press handling according to gesture consumption. This
also changes `Modifier.draggable`, which previously did not delay presses.

## Overscroll and anchored dragging

### Overscroll factories (1.8.0)

Replace `OverscrollConfiguration` and `LocalOverscrollConfiguration` with
`rememberPlatformOverscrollFactory` and `LocalOverscrollFactory`. Disable
overscroll with `LocalOverscrollFactory provides null`, or customize it with
`rememberPlatformOverscrollFactory(color, padding)`.

Scroll, lazy, grid, staggered-grid, and pager APIs accept a custom
`OverscrollEffect`. Use `withoutVisualEffect` and `withoutEventHandling` to
separate event processing from rendering across components. Never draw the
same effect twice.

### Anchored-draggable validation (1.8.0)

`AnchoredDraggableState.confirmValueChange` is deprecated. Remove disallowed
values from the active anchor set, and use an `OverscrollEffect` to signal
that the requested action is unavailable.

## Two-dimensional and area scrolling

### Two-axis state (1.9.0)

Use `Modifier.scrollable2D`, `Scrollable2DState`, its state factories, and
common scroll extensions for two-axis scrolling. The final
`Scrollable2DState.canScroll` contract takes an `Offset`, not an angle.

### Scrollable areas (1.10.0)

`Modifier.scrollableArea()` combines scrolling with bounds clipping and
derives content direction from orientation, RTL, and `reverseScrolling`.

`SnapFlingBehavior` permits an overshooting `snapAnimationSpec`, enabling
bouncy snap springs. Overshoot is still ignored in the approach phase.

Mouse-wheel scrolling handles two-dimensional deltas; tests can inject
matching deltas with `MouseInjectionScope`.

## Indicators, callbacks, and visibility

### Scroll notifications (1.9.0)

Compose can dispatch `ViewTreeObserver.OnScrollChanged` under
`isOnScrollChangedCallbackEnabled`; a modifier node can explicitly call
`DelegatableNode.dispatchOnScrollChanged`.

`Modifier.onFirstVisible` and `Modifier.onVisibilityChanged` support
impression logging, autoplay, and other visibility-driven work.

### Scroll state and custom indicators

In 1.10.0, `ScrollIndicatorState` represents scrollbar state via
`ScrollableState.scrollIndicatorState`. Implementations exist for
`ScrollState`, `LazyListState`, `LazyGridState`, `LazyStaggeredGridState`, and
`PagerState`.

In 1.11.0, draw custom indicators with `Modifier.scrollIndicator` and
`ScrollIndicatorFactory`.

### Visibility corrections (1.10.0 and 1.11.0)

In 1.10.0, `onVisibilityChanged` stops calling back for a node that is
initially invisible and correctly emits `false` after a nonzero
`minDurationMs`. `onVisibilityChangedNode()` exposes this behavior to custom
delegatable modifier nodes.

In 1.11.0, `Modifier.onFirstVisible()` is deprecated because it fires each
time an item becomes visible, not only once. Use `onVisibilityChanged()` and
track previous visibility according to the use case. Custom `Modifier.Node`
implementations can use the node-level visibility machinery.

`Modifier.visible` suppresses drawing while retaining occupied layout space.

### Frequently changing values (1.10.0)

`PagerState.currentPageOffsetFraction` and `ScrollState.value` are annotated
`@FrequentlyChangingValue`. Avoid direct composition reads when they would
cause unnecessary recomposition.

## Lazy layout infrastructure

### Custom layouts and prefetch (1.9.0)

`LazyLayout`, `LazyLayoutItemProvider`, and `LazyLayoutMeasureScope` are
stable, with a new `LazyLayoutMeasurePolicy`. The empty
`LazyLayoutPrefetchState` constructor and its precomposition and premeasure
scheduling methods are stable. Custom `PrefetchScheduler` is deprecated;
allow automatic internal scheduling.

### Beyond-bounds and key indexes (1.10.0)

Foundation adds `BeyondBoundsLayoutModifierNode` for focus-search layout and
`LazyLayoutKeyIndexMap` with a default implementation factory.

## Pointer and trackpad input

### Pointer hit expansion (1.8.0)

`PointerInputModifierNode.touchBoundsExpansion` enlarges the hit bounds of a
single pointer-input node. `BringIntoViewResponderModifierNode` provides a
node-level, platform-implementable bring-into-view mechanism.

### Trackpad events and tests (1.11.0)

Trackpad gestures that drive a cursor normally appear as mouse input.
Platform pan and scale gestures instead use `PanStart`, `PanMove`, `PanEnd`,
`ScaleStart`, `ScaleChange`, and `ScaleEnd` pointer event types.

Inject them with `SemanticsNodeInteraction.performTrackpadInput` or
`MultiModalInjectionScope.trackpad`. Multimodal key and rotary injection APIs
are stable, and pan/scale end injection accepts `delayMillis`.
