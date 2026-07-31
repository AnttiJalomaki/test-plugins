# Input, Focus, and Scrolling

## Focus

### Focus navigation and restoration (1.8.0)

Use receiver-based `FocusProperties.onEnter` and `onExit` instead of the former
`enter` and `exit` properties. `FocusRequester` and `FocusTargetModifierNode`
support `requestFocus(FocusDirection)`. `Modifier.focusRestorer()` takes a
non-null `FocusRequester` parameter named `fallback`, defaulting to
`FocusRequester.Default`; it no longer takes a fallback lambda.

### Pointer-down and child focus behavior (1.10.0)

Mouse or touchpad presses outside the focused node clear focus by default on
Android. Set `AbstractComposeView.isClearFocusOnPointerDownEnabled = false` to
opt out. A non-focusable `FocusTargetModifierNode.requestFocus()` routes to a
child. `DelegatableNode.requestFocusForChildInRootBounds()` finds an overlapping
child, while `ComposeUiFlags.isInitialFocusOnFocusableAvailable` enables
non-touch initial focus.

## Indications and delayed presses

### `IndicationNodeFactory` migration (1.9.0)

After recompiling against Foundation 1.9, no-explicit-indication overloads of
`clickable`, `combinedClickable`, `selectable`, `toggleable`, and
`triStateToggleable` require an `IndicationNodeFactory` from `LocalIndication`.
A deprecated `Indication` there can crash at runtime. Migrate it, use an
explicit-indication overload as a bridge, or—only on a version that still
contains it—temporarily disable
`ComposeFoundationFlags.isNonComposedClickableEnabled`.

### Removed interaction flags (1.10.0)

The non-composed-clickable bridge flag is removed, so the indication migration
can no longer be postponed. Flags controlling scroll callbacks, fling
continuation, drag pickup, automatic nested prefetch, pointer velocity, and
pointer/nested-scroll interoperability fixes were also removed. Delete those
assignments and rely on the finalized behavior.

### Delayed presses in drag containers (1.11.0)

`ComposeFoundationFlags.isDelayPressesUsingGestureConsumptionEnabled` opts
drag containers into delaying press handling according to gesture consumption.
It also affects `Modifier.draggable`, which previously did not delay presses.

## Gesture dispatch and pointer input

### Immediate tap dispatch and nested hand-off (1.8.0)

Tap detectors use immediate coroutine dispatch by default. On releases that
still expose the bridge, its name is
`ComposeFoundationFlags.isDetectTapGesturesImmediateCoroutineDispatchEnabled`.
A parent draggable or scrollable can take over a gesture abandoned by a child;
a fling reaching a bound hands remaining velocity to the next scrollable.

### Two-axis gestures (1.9.0)

Use `Modifier.scrollable2D`, `Scrollable2DState`, its state factories, and the
common scroll extensions for two-axis scrolling. The final `canScroll` contract
takes an `Offset`, not an angle. A `detectDragGestures` overload controls touch
slop and orientation locking. `ViewConfiguration.minimumFlingVelocity` exposes
the minimum fling threshold.

### Indirect-pointer rename (1.10.0)

All indirect-touch APIs were renamed to indirect-pointer counterparts. Update
imports, modifier calls, and event types together.

### Trackpad events and injection (1.11.0)

Cursor-driving trackpad gestures generally appear as mouse input. Platform pan
and scale gestures use `PanStart`, `PanMove`, `PanEnd`, `ScaleStart`,
`ScaleChange`, and `ScaleEnd`. Tests can inject them with
`performTrackpadInput` or `MultiModalInjectionScope.trackpad`. Multimodal key
and rotary injection APIs are stable; pan/scale end injection accepts
`delayMillis`.

## Overscroll and draggable state

### Overscroll factories and split ownership (1.8.0)

Replace `OverscrollConfiguration` and `LocalOverscrollConfiguration` with
`rememberPlatformOverscrollFactory` and `LocalOverscrollFactory`.

- Disable overscroll with `LocalOverscrollFactory provides null`.
- Customize it with `rememberPlatformOverscrollFactory(color, padding)`.
- Supply an `OverscrollEffect` directly to scroll, lazy, grid,
  staggered-grid, and pager APIs.
- Use `withoutVisualEffect` and `withoutEventHandling` when separate components
  own input and rendering. Never render the same effect twice.

### Anchored-draggable validation (1.8.0)

`AnchoredDraggableState.confirmValueChange` is deprecated. Omit disallowed
values from the current anchor set. An `OverscrollEffect` can communicate that
the requested action is unavailable.

## Scrolling and lazy layout infrastructure

### Custom lazy layouts and prefetch (1.9.0)

`LazyLayout`, `LazyLayoutItemProvider`, and `LazyLayoutMeasureScope` are stable,
along with `LazyLayoutMeasurePolicy`. The empty `LazyLayoutPrefetchState`
constructor and its precomposition and premeasure scheduling methods are stable.
Custom `PrefetchScheduler` implementations are deprecated; use automatic
internal scheduling.

### Scroll indicators and scrollable areas (1.10.0)

`ScrollIndicatorState` describes scrollbar position through
`ScrollableState.scrollIndicatorState`. Implementations exist for
`ScrollState`, `LazyListState`, `LazyGridState`, `LazyStaggeredGridState`, and
`PagerState`. `Modifier.scrollableArea()` combines scrolling and clipping and
derives content direction from orientation, RTL, and `reverseScrolling`.

### Scrolling and selection behavior (1.10.0)

`SnapFlingBehavior` permits an overshooting `snapAnimationSpec`, enabling a
bouncy snap while ignoring overshoot during approach. Mouse wheels support
two-dimensional deltas, with matching `MouseInjectionScope` support. Double-tap
word selection works in `SelectionContainer` and value/callback `BasicTextField`.

### Frequently changing values (1.10.0)

`PagerState.currentPageOffsetFraction` and `ScrollState.value` carry
`@FrequentlyChangingValue`, so the corresponding lint rule treats direct
composition reads as frequently invalidating state.

### Scroll indicators and draw visibility (1.11.0)

Render custom indicators with `Modifier.scrollIndicator` and
`ScrollIndicatorFactory`. `Modifier.visible` suppresses drawing while retaining
the composable's layout space.

## Visibility callbacks

### Scroll and visibility notifications (1.9.0)

Compose can dispatch `ViewTreeObserver.OnScrollChanged` when its then-current
scroll callback bridge is enabled, and a `DelegatableNode` can call
`dispatchOnScrollChanged`. `Modifier.onFirstVisible` and
`Modifier.onVisibilityChanged` support impression logging and autoplay.

### Corrected visibility behavior (1.10.0)

`onVisibilityChanged` no longer calls back for an initially invisible node and
correctly reports `false` after a nonzero `minDurationMs`.
`onVisibilityChangedNode()` exposes the machinery to a custom delegatable
modifier node.

### `onFirstVisible` deprecation (1.11.0)

`Modifier.onFirstVisible()` is deprecated because it can fire each time an item
becomes visible, not literally only once. Use `onVisibilityChanged()` and track
prior visibility according to the product requirement.

## Removed flags in newer releases

### Foundation, UI, and Runtime cleanup (1.11.0)

Delete assignments to removed Foundation flags:

- `isDetectTapGesturesImmediateCoroutineDispatchEnabled`
- `isNonSuspendingPointerInputInClickableEnabled`
- `isTextFieldDpadNavigationEnabled`
- `isKeepInViewFocusObservationChangeEnabled`

Also remove the UI flag `isSemanticAutofillEnabled` and Runtime flags
`isMovingNestedMovableContentEnabled` and
`isMovableContentUsageTrackingEnabled`. Semantic autofill, text-field D-pad
navigation, and keep-in-view focus observation are always enabled.
