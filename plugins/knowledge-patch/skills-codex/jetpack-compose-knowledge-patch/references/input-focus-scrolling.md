# Input, Focus, and Scrolling

## Semantic autofill

Semantics-based autofill arrived in 1.8.0. Text autofill requires both Compose
UI and Foundation at 1.8 or newer. The legacy autofill APIs are deprecated,
and several contracts changed:

- `AutofillManager` is an abstract class.
- `InputText` exposes the value before output transformation.
- `requestAutofill` is no longer a method on the manager.
- The text toolbar can initiate autofill.
- `LocalAutofillHighlightColor` uses `Color`.

Typed autofill in 1.10.0 expands `FillableData` to text, Boolean, integer, list,
and date values. Factories moved to the companion object; create data with
`FillableData.createFrom(value)` and read a date through `dateMillisValue`.
Replace deprecated `onAutofillText` with the `fillableData` semantics property
and `onFillData` action. A composition local can customize the highlight brush
shown after a successful fill.

In 1.11.0, the `isSemanticAutofillEnabled` UI flag is removed because semantic
autofill is always enabled. Delete flag assignments rather than trying to
restore legacy autofill behavior.

## Focus navigation and restoration

Stable focus APIs in 1.8.0 replace `FocusProperties.enter` and `exit` with
receiver-based `onEnter` and `onExit`. `FocusRequester` and
`FocusTargetModifierNode` support `requestFocus(FocusDirection)`.

`Modifier.focusRestorer()` takes a non-null `FocusRequester` parameter named
`fallback`, defaulting to `FocusRequester.Default`; it no longer takes a
lambda.

Further focus behavior in 1.10.0:

- A mouse or touchpad press outside the focused node clears focus by default.
  Set `AbstractComposeView.isClearFocusOnPointerDownEnabled = false` to opt out.
- Calling `FocusTargetModifierNode.requestFocus()` on a non-focusable target
  routes focus to a child.
- `DelegatableNode.requestFocusForChildInRootBounds()` targets a child that
  overlaps the supplied root bounds.
- `ComposeUiFlags.isInitialFocusOnFocusableAvailable` can enable non-touch
  initial focus.

The text-field D-pad navigation and keep-in-view focus-observation behavior
are always enabled in 1.11.0. Delete assignments to the removed
`ComposeFoundationFlags.isTextFieldDpadNavigationEnabled` and
`isKeepInViewFocusObservationChangeEnabled` flags.

## Haptic feedback

`LocalHapticFeedback` supplies a default Android implementation when the
vibrator reports support (since 1.8.0). Available `HapticFeedbackType` values
include `Confirm`, `ContextClick`, `GestureEnd`, `GestureThresholdActivate`,
`Reject`, `SegmentFrequentTick`, `SegmentTick`, `ToggleOn`, `ToggleOff`, and
`VirtualKey`.

## IndicationNodeFactory migration

After recompilation against 1.9.0, the `clickable`, `combinedClickable`,
`selectable`, `toggleable`, and `triStateToggleable` overloads without an
explicit `Indication` only accept an `IndicationNodeFactory` supplied through
`LocalIndication`. Supplying a deprecated `Indication` can crash at runtime.

Migrate the indication implementation or use an overload with an explicit
indication as a compatibility bridge. In 1.9, code could temporarily set
`ComposeFoundationFlags.isNonComposedClickableEnabled = false`; that escape
hatch is removed in 1.10.0 and assignments must be deleted.

## Overscroll and anchored dragging

Replace `OverscrollConfiguration` and `LocalOverscrollConfiguration` with
`rememberPlatformOverscrollFactory` and `LocalOverscrollFactory` (since
1.8.0). Disable overscroll by providing `null`, or customize it by providing a
factory:

```kotlin
CompositionLocalProvider(
    LocalOverscrollFactory provides
        rememberPlatformOverscrollFactory(color, padding)
) {
    content()
}
```

Scroll, lazy, grid, staggered-grid, and pager APIs accept a custom
`OverscrollEffect`. `withoutVisualEffect` and `withoutEventHandling` allow
event handling and drawing to live in separate components. Never draw the
same effect from both components.

`AnchoredDraggableState.confirmValueChange` is deprecated. Remove forbidden
values from the active anchor set, and use an `OverscrollEffect` to signal that
the requested action is unavailable.

## Gesture dispatch and nested hand-off

Tap gesture detectors use immediate coroutine dispatch by default in 1.8.0;
the compatibility switch was
`ComposeFoundationFlags.isDetectTapGesturesImmediateCoroutineDispatchEnabled`.
That flag is removed in 1.11.0, so delete assignments.

Also delete 1.11.0 assignments to the removed
`ComposeFoundationFlags.isNonSuspendingPointerInputInClickableEnabled` flag.

A parent draggable or scrollable can pick up a gesture that a child abandons,
and a fling reaching a bound hands remaining velocity to the next scrollable
in the chain. The flags for fling continuation and drag pickup are removed in
1.10.0, along with flags for pointer-velocity adjustment and pointer/nested-
scroll interop fixes; the corrected behaviors no longer need opt-ins.

The 1.9.0 `detectDragGestures` overload exposes touch-slop and orientation-lock
control. `ViewConfiguration.minimumFlingVelocity` exposes the lower fling
threshold.

`ComposeFoundationFlags.isDelayPressesUsingGestureConsumptionEnabled` in
1.11.0 lets drag containers delay press handling according to gesture
consumption. This changes `Modifier.draggable`, which did not previously delay
presses.

## Two-dimensional scrolling

`Modifier.scrollable2D`, `Scrollable2DState`, its state factories, and common
scroll extensions add two-axis scrolling in 1.9.0. The finalized
`Scrollable2DState.canScroll` contract receives an `Offset`, not an angle.

In 1.10.0, mouse-wheel scrolling understands two-dimensional deltas, with
matching `MouseInjectionScope` test support. `Modifier.scrollableArea()`
combines scrolling with bounds clipping and derives content direction from
orientation, RTL, and `reverseScrolling`.

`SnapFlingBehavior` accepts an overshooting `snapAnimationSpec`, allowing a
bouncy snap spring; overshoot is still ignored during the approach phase.

## Scroll indicators

`ScrollableState.scrollIndicatorState` exposes a `ScrollIndicatorState` in
1.10.0. Implementations exist for `ScrollState`, `LazyListState`,
`LazyGridState`, `LazyStaggeredGridState`, and `PagerState`.

For a custom indicator, use `Modifier.scrollIndicator` and
`ScrollIndicatorFactory` (since 1.11.0). Material 3's scrollbar parameter is a
separate component contract and is non-nullable in
material3-1.5.0-alpha24.

## Scroll and visibility callbacks

Compose 1.9.0 can dispatch `ViewTreeObserver.OnScrollChanged` under
`isOnScrollChangedCallbackEnabled`; modifier nodes can explicitly call
`DelegatableNode.dispatchOnScrollChanged`. That temporary behavior flag is
removed in 1.10.0, so remove assignments to it.

`Modifier.onFirstVisible` and `Modifier.onVisibilityChanged` were introduced
for list impressions, autoplay, and related work. In 1.10.0,
`onVisibilityChanged` no longer calls back for an initially invisible node and
correctly emits `false` after a nonzero `minDurationMs`.
`onVisibilityChangedNode()` exposes the mechanism for a delegatable custom
`Modifier.Node`.

As of 1.11.0, `Modifier.onFirstVisible()` is deprecated because it fires each
time an item becomes visible, not just once. Use `onVisibilityChanged()` and
store prior visibility explicitly when the event must be one-shot.

## Indirect pointer and trackpad input

All indirect-touch APIs are renamed to indirect-pointer equivalents in
1.10.0.

In 1.11.0, trackpad gestures that drive a cursor generally arrive as mouse
input. Platform pan and scale gestures use `PanStart`, `PanMove`, `PanEnd`,
`ScaleStart`, `ScaleChange`, and `ScaleEnd` pointer-event types. Tests can
inject these through `SemanticsNodeInteraction.performTrackpadInput` or
`MultiModalInjectionScope.trackpad`. Multimodal key and rotary injection APIs
are stable, and pan/scale end injection accepts `delayMillis`.
