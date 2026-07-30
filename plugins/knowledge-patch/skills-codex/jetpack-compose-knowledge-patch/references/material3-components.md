# Material 3 Components

## Dependencies, icons, and ripple

Material 3 stops adding `material-icons-core` transitively in
material3-1.4.0. Declare it explicitly if existing source still imports it.
The `androidx.compose.material.icons` library is no longer updated or
recommended; prefer a Material Symbols Vector Drawable XML downloaded from
the Android tab of the Material icons site.

In material3-1.5.0-alpha24, the new
`androidx.compose.material3:material3-ripple` library replaces direct use of
`material-ripple` and renders focus with inset rings rather than an opacity
layer. Apps using `material3` should retain its built-in themed ripple
configuration. Depend on the new artifact directly only when replacing a
direct `material-ripple` dependency or adding inset focus rings without the
full Material 3 library.

```kotlin
dependencies {
    implementation("androidx.compose.material3:material3-ripple:1.5.0-alpha24")
}
```

## Theme motion and navigation colors

Material 3 components obtain motion from a `MotionScheme` supplied through
`MaterialTheme` in material3-1.4.0. Create the standard scheme with
`MotionScheme.standard()`. A modifier node can read it with:

```kotlin
currentValueOf(MotionTheme.LocalMotionScheme)
```

Selected labels in `NavigationBarItem` and `NavigationRailItem` use
`MaterialTheme.colorScheme.secondary` instead of `onSurface`. To preserve the
old appearance, copy the default colors and set `selectedTextColor` to
`MaterialTheme.colorScheme.onSurface`.

## Expressive and override APIs

Public APIs still annotated `ExperimentalMaterial3ExpressiveApi` or
`ExperimentalMaterial3ComponentOverrideApi` were removed from the stable
material3-1.4.0 line. Code using them must move to a 1.5.0 alpha artifact.

The expressive `TimePicker` adds a scroll-style variant in
material3-1.5.0-alpha24.

## State-backed and secure text fields

`TextField` and `OutlinedTextField` have `TextFieldState` overloads in
material3-1.4.0. Their `TextFieldDecorator`-compatible decoration APIs are
stable. `labelPosition` can keep the label minimized, allowing a placeholder
to remain visible while the field is unfocused.

Use `SecureTextField` or `OutlinedSecureTextField` for Material password
entry.

## Search bars and app bars

The search UI is split into collapsed and expanded surfaces:

- `SearchBar` is the collapsed surface.
- `ExpandedFullScreenSearchBar` and `ExpandedDockedSearchBar` open expanded
  content in a new window.
- Drive the surfaces with `SearchBarState`.
- `TopSearchBar` adds inset and scroll handling.
- `InputField` has a state-backed overload.

In material3-1.5.0-alpha24, `SearchBarState` and the slot-based `SearchBar`
APIs are stable. Overloads driven by `expanded` and `onExpandedChange` are
deprecated; migrate to state-backed calls.

`AppBarWithSearch` is annotated with `@ExperimentalMaterial3Api` again in
material3-1.5.0-alpha24. Restore the opt-in at upgrading call sites.

## Tab rows

`TabRow` and `ScrollableTabRow` are deprecated in 1.9.0. Migrate to the
appropriate primary or secondary tab-row variant for the intended hierarchy.

## Time pickers

`TimePickerDialog` can host `TimePicker`, `TimeInput`, or a UI that switches
between them. Replace `TimePickerState.isAfternoon` with the `isPm` extension
property (material3-1.4.0).

## Date pickers

`DatePicker`, `DateRangePicker`, and their supporting APIs are stable in
material3-1.4.0. State factories and state extensions accept `LocalDate` and
`YearMonth` on API 26 or newer, or on older Android versions with desugaring.

- `getDisplayedMonth()` is non-null.
- The input-mode focus option accepts an optional `FocusRequester`, not a
  Boolean.
- Changing the state locale directly does not localize the default title or
  headline text; supply localized display content separately.

## Carousels

`HorizontalCenteredHeroCarousel` provides a center-aligned hero layout in
material3-1.4.0. Carousel composables accept `userScrollEnabled`.
`CarouselState` exposes `currentItem` and programmatic scrolling.

## Pull to refresh

Rename `PullToRefreshDefaults` properties as follows:

| Old | New |
| --- | --- |
| `shape` | `indicatorShape` |
| `containerColor` | `indicatorContainerColor` |

`indicatorMaxDistance` is also available. A custom `PullToRefreshState` must
implement `isAnimating`; it no longer inherits a default implementation.

## Tooltip positioning and dismissal

Use `rememberTooltipPositionProvider`, not the deprecated plain or rich
position-provider functions. `TooltipScope.layoutCoordinates` exposes the
anchor and supersedes `drawCaret`; custom caret shapes and positions are
supported.

The newer dismissal overload takes `onDismissRequest`, while `onDismiss` is
no longer suspend. `TooltipBox` defaults `focusable` to `false` and adds
`hasAction`. Plain and rich tooltips default to maximum widths of 200 dp and
320 dp, respectively.

## Bottom sheets

`ModalBottomSheet` adds `sheetGestureEnabled` in material3-1.4.0.
`ModalBottomSheetProperties` can prevent scrim clicks from requesting
dismissal; its light status-bar and navigation-bar options are Android-only.

`SheetState.isAnimationRunning` is public. The density-taking constructor is
deprecated in favor of positional and velocity thresholds.
`BottomSheetDefaults.windowInsets` includes `WindowInsets.safeDrawing.Top`.

## Wide navigation rails and short navigation bars

`WideNavigationRail`, `ShortNavigationBar`, and `NavigationItem` are stable in
material3-1.4.0. Migration details:

- `WideNavigationRailItem` requires `railExpanded`.
- `WideNavigationRailState` exposes Boolean current and target values.
- Use `Arrangement.Vertical` instead of `WideNavigationRailArrangement`.
- Use shape defaults under `WideNavigationRailDefaults`;
  `ModalWideNavigationRailDefaults` was removed.

## Navigation suites

`NavigationSuite`, `NavigationSuiteItem`, `NavigationSuiteColors`, and
`NavigationSuiteTypes` support additional layouts selected through
`navigationSuiteType`. The related scaffold and layout APIs accept optional
primary-action content.

## Slider state and layouts

Hoist slider state with `rememberSliderState` and range state with
`rememberRangeSliderState`. `SliderState.shouldAutoSnap` can disable automatic
snapping for custom animation, and `onValueChange` is public.

Material 3 also provides `VerticalSlider`, the center-origin `CenteredTrack`,
customizable external track corners and icons, and `trackCornerSize` for range
slider tracks.

## Color schemes and annotated links

The `ColorScheme` constructor without fixed color roles is deprecated, and the
constructor without surface-container roles is hidden. Custom schemes should
supply both role families.

`ColorScheme.contentColorFor(surfaceDim)` resolves to `onSurface`. Links in
`Text(AnnotatedString)` receive Material styling by default.

## Insets and display cutouts

Inset-aware Material 2 and Material 3 components include `displayCutout` in
their default `WindowInsets` in material3-1.4.0. Override the component's
inset argument if this avoidance is undesirable.

## Scrollbar state

The Material 3 scrollbar's `ScrollIndicatorState` parameter becomes
non-nullable in material3-1.5.0-alpha24. Supply a state instead of passing
`null`.
