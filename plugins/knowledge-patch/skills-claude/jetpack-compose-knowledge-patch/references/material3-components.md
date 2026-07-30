# Material 3 Components

## Manage icons and ripple dependencies

Material 3 no longer adds `material-icons-core` transitively. Declare that
artifact explicitly while legacy icons remain, but prefer a Material Symbols
Vector Drawable XML downloaded from the Android tab of the Material icons
site. The `androidx.compose.material.icons` library is no longer updated or
recommended (`material3-1.4.0`).

The `androidx.compose.material3:material3-ripple` artifact replaces direct use
of `material-ripple` and draws focus as inset rings rather than an opacity
layer. Apps using the full `material3` library should keep its built-in themed
ripple configuration. Depend on the ripple artifact directly only to replace a
direct `material-ripple` dependency or to add inset focus rings without the
full library (`material3-1.5.0-alpha24`).

```kotlin
dependencies {
    implementation(
        "androidx.compose.material3:material3-ripple:1.5.0-alpha24"
    )
}
```

## Account for navigation-item color

Selected `NavigationBarItem` and `NavigationRailItem` labels use
`MaterialTheme.colorScheme.secondary` instead of `onSurface`. To preserve the
old appearance, copy the default colors and set `selectedTextColor` to
`MaterialTheme.colorScheme.onSurface` (`material3-1.4.0`).

`TabRow` and `ScrollableTabRow` are deprecated. Select the matching primary or
secondary tab-row variant instead (1.9.0).

## Supply a motion scheme

Material 3 components take motion from a `MotionScheme` supplied through
`MaterialTheme`. Modifier nodes can read the current scheme with
`currentValueOf(MotionTheme.LocalMotionScheme)`. Construct the standard scheme
with `MotionScheme.standard()` (`material3-1.4.0`).

## Choose the matching stability line

Public APIs still annotated `ExperimentalMaterial3ExpressiveApi` or
`ExperimentalMaterial3ComponentOverrideApi` were removed from the stable
Material 3 1.4 line. Code that needs those APIs must use the 1.5 alpha line
(`material3-1.4.0`).

`AppBarWithSearch` is annotated with `@ExperimentalMaterial3Api` again. Restore
the opt-in at upgrading call sites (`material3-1.5.0-alpha24`).

## Use state-backed and secure text fields

`TextField` and `OutlinedTextField` have `TextFieldState` overloads, and their
`TextFieldDecorator`-compatible decoration APIs are stable. `labelPosition`
can keep a label minimized so an unfocused field can continue displaying a
placeholder. Use `SecureTextField` or `OutlinedSecureTextField` for password
entry (`material3-1.4.0`).

## Build search as collapsed and expanded surfaces

The collapsed `SearchBar` is separate from
`ExpandedFullScreenSearchBar` and `ExpandedDockedSearchBar`, which open the
expanded UI in a new window. Drive all of them with `SearchBarState`.
`TopSearchBar` handles insets and scrolling, and `InputField` has a state-backed
overload (`material3-1.4.0`).

`SearchBarState` and slot-based `SearchBar` APIs are stable. The older
`expanded`/`onExpandedChange` overloads are deprecated; migrate them to the
state-backed form (`material3-1.5.0-alpha24`).

## Select time-picker presentation

`TimePickerDialog` can host `TimePicker`, `TimeInput`, or a UI that switches
between them. Replace `TimePickerState.isAfternoon` with the `isPm` extension
property (`material3-1.4.0`).

The expressive `TimePicker` also offers a scroll variant as an alternative to
its other input styles (`material3-1.5.0-alpha24`).

## Configure carousels

`HorizontalCenteredHeroCarousel` supplies a center-aligned hero layout.
Carousel composables accept `userScrollEnabled`; `CarouselState` exposes
`currentItem` and programmatic scrolling (`material3-1.4.0`).

## Migrate pull to refresh

Within `PullToRefreshDefaults`, rename `shape` to `indicatorShape` and
`containerColor` to `indicatorContainerColor`. `indicatorMaxDistance` is also
available. A custom `PullToRefreshState` must implement `isAnimating` instead
of relying on a default implementation (`material3-1.4.0`).

## Use date-picker state accurately

`DatePicker`, `DateRangePicker`, and their supporting APIs are stable. State
factories and state extensions support `LocalDate` and `YearMonth` on API 26+
or with desugaring, and `getDisplayedMonth()` is non-null
(`material3-1.4.0`).

The input-mode focus option accepts an optional `FocusRequester` instead of a
Boolean. Directly setting a state locale does not localize the default title
or headline text (`material3-1.4.0`).

## Position and dismiss tooltips

Use `rememberTooltipPositionProvider` instead of the deprecated plain and rich
position-provider functions. `TooltipScope.layoutCoordinates` exposes the
anchor and supersedes `drawCaret`; custom caret shapes and positions are
supported (`material3-1.4.0`).

Use the dismissal overload with `onDismissRequest`. `onDismiss` is no longer
suspend. `TooltipBox` defaults `focusable` to `false` and adds `hasAction`.
Plain and rich tooltips default to maximum widths of 200 dp and 320 dp,
respectively (`material3-1.4.0`).

## Control modal bottom sheets

`ModalBottomSheet` adds `sheetGestureEnabled`.
`ModalBottomSheetProperties` can prevent scrim clicks from requesting
dismissal; its light status- and navigation-bar options are Android-only
(`material3-1.4.0`).

`SheetState.isAnimationRunning` is public. Its density constructor is
deprecated in favor of positional and velocity thresholds.
`BottomSheetDefaults.windowInsets` includes `WindowInsets.safeDrawing.Top`
(`material3-1.4.0`).

## Build wide and short navigation

`WideNavigationRail`, `ShortNavigationBar`, and `NavigationItem` are stable.
`WideNavigationRailItem` requires `railExpanded`; `WideNavigationRailState`
exposes Boolean current and target values (`material3-1.4.0`).

Use `Arrangement.Vertical` instead of `WideNavigationRailArrangement`. Use the
renamed shape defaults under `WideNavigationRailDefaults`; the
`ModalWideNavigationRailDefaults` object was removed
(`material3-1.4.0`).

## Select a navigation-suite layout

`NavigationSuite`, `NavigationSuiteItem`, `NavigationSuiteColors`, and
`NavigationSuiteTypes` support extra navigation layouts selected through
`navigationSuiteType`. The corresponding scaffold and layout APIs accept
optional primary-action content (`material3-1.4.0`).

## Hoist slider state and customize tracks

Use `rememberSliderState` and `rememberRangeSliderState`.
`SliderState.shouldAutoSnap` can disable automatic snapping for custom
animation, and `onValueChange` is public (`material3-1.4.0`).

Available layouts and customization include `VerticalSlider`, a center-origin
`CenteredTrack`, external track corners and icons, and `trackCornerSize` for
range-slider tracks (`material3-1.4.0`).

## Supply complete color roles

The `ColorScheme` constructor without fixed color roles is deprecated, and the
one without surface-container roles is hidden. Custom schemes should provide
both role families. `ColorScheme.contentColorFor(surfaceDim)` resolves to
`onSurface`. Links in `Text(AnnotatedString)` receive Material styling by
default (`material3-1.4.0`).

## Handle component insets

Inset-aware Material 2 and Material 3 components include `displayCutout` in
their default `WindowInsets`. Override a component's inset parameter when the
new avoidance is not wanted (`material3-1.4.0`).

## Always supply scrollbar state

The Material 3 scrollbar `ScrollIndicatorState` parameter is non-nullable.
Callers must supply an actual state instead of `null`
(`material3-1.5.0-alpha24`).
