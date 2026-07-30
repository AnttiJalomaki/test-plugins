# Input, Scrolling, and Focus

## Complete the indication migration

After recompilation against Foundation 1.9, indication-less overloads of
`clickable`, `combinedClickable`, `selectable`, `toggleable`, and
`triStateToggleable` only accept an `IndicationNodeFactory` supplied through
`LocalIndication`. Supplying a deprecated `Indication` can crash at runtime
(1.9.0).

Migrate the indication, use an overload with an explicit indication as a
compatibility bridge, or temporarily set
`ComposeFoundationFlags.isNonComposedClickableEnabled = false` only while on
Foundation 1.9. The flag is removed in 1.10.0, so the migration cannot be
deferred after that upgrade.

## Configure overscroll through factories

Replace `OverscrollConfiguration` and `LocalOverscrollConfiguration` with
`rememberPlatformOverscrollFactory` and `LocalOverscrollFactory`. Supply
`LocalOverscrollFactory provides null` to disable overscroll, or provide
`rememberPlatformOverscrollFactory(color, padding)` to customize it (1.8.0).

Scroll, lazy, grid, staggered-grid, and pager APIs accept a custom
`OverscrollEffect`. `withoutVisualEffect` and `withoutEventHandling` let input
handling and rendering occur in separate components. Do not draw the same
effect twice (1.8.0).

## Validate anchored drag through anchors

`AnchoredDraggableState.confirmValueChange` is deprecated. Exclude disallowed
values from the active anchors and use an `OverscrollEffect` to communicate
that the attempted action is unavailable (1.8.0).

## Hand gestures and flings to parents

Tap detectors use immediate coroutine dispatch by default. The compatibility
switch moved to
`ComposeFoundationFlags.isDetectTapGesturesImmediateCoroutineDispatchEnabled`
(1.8.0), then was removed in 1.11.0.

A parent draggable or scrollable can take over a gesture abandoned by a child.
When a fling reaches a bound, remaining velocity is passed to the next
scrollable in the chain (1.8.0).

`ComposeFoundationFlags.isDelayPressesUsingGestureConsumptionEnabled` opts
drag containers into delaying presses according to gesture consumption. This
also changes `Modifier.draggable`, which did not previously delay presses
(1.11.0).

## Scroll on two axes

Use `Modifier.scrollable2D`, `Scrollable2DState`, its state factories, and the
common scroll extensions for two-axis scrolling. The final
`Scrollable2DState.canScroll` contract accepts an `Offset`, not an angle
(1.9.0).

A `detectDragGestures` overload controls touch slop and orientation locking.
`ViewConfiguration.minimumFlingVelocity` exposes the lower fling threshold
(1.9.0).

Mouse-wheel scrolling handles two-dimensional deltas and can be tested through
matching `MouseInjectionScope` support (1.10.0).

## Observe and dispatch scrolling

Compose can dispatch `ViewTreeObserver.OnScrollChanged` when
`isOnScrollChangedCallbackEnabled` is active. A modifier node can call
`DelegatableNode.dispatchOnScrollChanged` explicitly (1.9.0). The temporary
on-scroll behavior flag is removed in 1.10.0.

## Build scrollable areas and indicators

`Modifier.scrollableArea()` combines scrolling with bounds clipping and
derives content direction from orientation, RTL, and `reverseScrolling`
(1.10.0).

`ScrollIndicatorState` represents scrollbar state through
`ScrollableState.scrollIndicatorState`. Implementations exist for
`ScrollState`, `LazyListState`, `LazyGridState`, `LazyStaggeredGridState`, and
`PagerState` (1.10.0).

Use `Modifier.scrollIndicator` and `ScrollIndicatorFactory` to render a custom
indicator (1.11.0).

## Tune snapping and selection

`SnapFlingBehavior` accepts an overshooting `snapAnimationSpec`, which permits
bouncy snap springs while still ignoring overshoot in the approach phase
(1.10.0).

Double-tap word selection works in `SelectionContainer` and in the
value/on-value-change `BasicTextField` (1.10.0).

## Navigate and restore focus

Stable focus APIs replace `FocusProperties.enter` and `exit` with
receiver-based `onEnter` and `onExit`. `FocusRequester` and
`FocusTargetModifierNode` add `requestFocus(FocusDirection)` (1.8.0).

`Modifier.focusRestorer()` now takes a non-null `FocusRequester` parameter
named `fallback`, defaulting to `FocusRequester.Default`, rather than a lambda
(1.8.0).

Mouse or touchpad presses outside the focused node clear focus by default. Set
`AbstractComposeView.isClearFocusOnPointerDownEnabled = false` to opt out
(1.10.0).

A non-focusable `FocusTargetModifierNode.requestFocus()` routes focus to a
child. `DelegatableNode.requestFocusForChildInRootBounds()` targets a child
overlapping the root bounds. Enable non-touch initial focus with
`ComposeUiFlags.isInitialFocusOnFocusableAvailable` (1.10.0).

## Handle indirect pointers and trackpads

All indirect-touch APIs were renamed to indirect-pointer counterparts
(1.10.0).

Trackpad gestures that drive a cursor generally arrive as mouse input.
Platform pan and scale gestures use `PanStart`, `PanMove`, `PanEnd`,
`ScaleStart`, `ScaleChange`, and `ScaleEnd` pointer event types (1.11.0).

Tests can inject these gestures through
`SemanticsNodeInteraction.performTrackpadInput` or
`MultiModalInjectionScope.trackpad`. Multimodal key and rotary injection APIs
are stable, and pan/scale end injection accepts `delayMillis` (1.11.0).

## Use expanded haptic types

`LocalHapticFeedback` supplies a default Android implementation when vibrator
support is reported. Available additions are `Confirm`, `ContextClick`,
`GestureEnd`, `GestureThresholdActivate`, `Reject`, `SegmentFrequentTick`,
`SegmentTick`, `ToggleOn`, `ToggleOff`, and `VirtualKey` (1.8.0).

## Remove expired input behavior flags

From 1.10.0, delete assignments to removed flags for on-scroll callbacks,
fling continuation, drag pickup, automatic nested prefetch, pointer-velocity
adjustment, and pointer/nested-scroll interop fixes. Their migrated behaviors
no longer use temporary switches.

From 1.11.0, delete assignments to these removed Foundation flags:

- `isDetectTapGesturesImmediateCoroutineDispatchEnabled`
- `isNonSuspendingPointerInputInClickableEnabled`
- `isTextFieldDpadNavigationEnabled`
- `isKeepInViewFocusObservationChangeEnabled`

Text-field D-pad navigation and keep-in-view focus observation are always
enabled after those removals.
