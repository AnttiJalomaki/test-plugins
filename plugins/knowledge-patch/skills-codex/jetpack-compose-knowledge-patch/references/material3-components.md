# Material 3 Components

## Dependencies, theme, and color

### Material Icons dependency removal (material3-1.4.0)

Material 3 no longer adds `material-icons-core` transitively. Existing code
that still imports it must declare it directly. The
`androidx.compose.material.icons` library is no longer updated or recommended;
prefer a Material Symbols Vector Drawable XML downloaded from the Android tab
of the Material icons site.

### Navigation-item selected labels (material3-1.4.0)

Selected labels in `NavigationBarItem` and `NavigationRailItem` use
`MaterialTheme.colorScheme.secondary` instead of `onSurface`. To keep the old
look, copy the defaults and set `selectedTextColor` to
`MaterialTheme.colorScheme.onSurface`.

### Motion schemes (material3-1.4.0)

Material 3 components take motion from a `MotionScheme` supplied through
`MaterialTheme`. Modifier nodes can read it with
`currentValueOf(MotionTheme.LocalMotionScheme)`. Create the standard scheme
with `MotionScheme.standard()`.

### Stable-line expressive API removal (material3-1.4.0)

Public APIs still annotated `ExperimentalMaterial3ExpressiveApi` or
`ExperimentalMaterial3ComponentOverrideApi` were removed from the stable 1.4
line. Code requiring those APIs must intentionally select the corresponding
Material 3 alpha line; do not expect an opt-in to restore them on stable 1.4.

### Color-scheme constructor requirements (material3-1.4.0)

The `ColorScheme` constructor lacking fixed color roles is deprecated, and the
constructor lacking surface-container roles is hidden. Custom schemes should
supply both role families. `contentColorFor(surfaceDim)` resolves to
`onSurface`. Links in `Text(AnnotatedString)` receive Material styling by
default.

## Text fields and search

### State-backed and secure text fields (material3-1.4.0)

`TextField` and `OutlinedTextField` have `TextFieldState` overloads, and their
`TextFieldDecorator`-compatible decoration APIs are stable. `labelPosition`
can keep the label minimized so an unfocused field still shows its placeholder.
Use `SecureTextField` or `OutlinedSecureTextField` for password entry.

### Search-bar state and expanded views (material3-1.4.0)

`SearchBar` renders the collapsed state. Use
`ExpandedFullScreenSearchBar` or `ExpandedDockedSearchBar` for the expanded UI,
which opens in a new window. Drive both with `SearchBarState`. `TopSearchBar`
adds inset and scroll handling, and `InputField` offers a state-backed overload.

## Navigation

### Tab-row migration (1.9.0)

`TabRow` and `ScrollableTabRow` are deprecated. Choose the primary or secondary
tab-row variant according to the tab hierarchy and visual treatment.

### Wide navigation rails (material3-1.4.0)

`WideNavigationRail`, `ShortNavigationBar`, and `NavigationItem` are stable.
`WideNavigationRailItem` requires `railExpanded`, and
`WideNavigationRailState` exposes Boolean current and target values. Replace
`WideNavigationRailArrangement` with `Arrangement.Vertical`. Use the renamed
shape defaults on `WideNavigationRailDefaults`; `ModalWideNavigationRailDefaults`
was removed.

### Navigation-suite layouts (material3-1.4.0)

`NavigationSuite`, `NavigationSuiteItem`, `NavigationSuiteColors`, and
`NavigationSuiteTypes` support different navigation forms selected with
`navigationSuiteType`. Their scaffold and layout APIs accept optional
primary-action content.

## Pickers

### Time-picker APIs (material3-1.4.0)

`TimePickerDialog` can contain `TimePicker`, `TimeInput`, or a UI switching
between them. Replace `TimePickerState.isAfternoon` with the `isPm` extension
property.

### Date-picker expansion (material3-1.4.0)

`DatePicker`, `DateRangePicker`, and supporting APIs are stable. Their state
factories and extensions support `LocalDate` and `YearMonth` on API 26 or newer,
or with desugaring. `getDisplayedMonth()` is non-null.

The input-mode focus option takes an optional `FocusRequester`, not a Boolean.
Changing the state locale does not localize the default title or headline;
supply localized content explicitly when changing locale programmatically.

## Carousel, refresh, and sliders

### Hero carousel and state (material3-1.4.0)

`HorizontalCenteredHeroCarousel` provides a center-aligned hero layout.
Carousel composables accept `userScrollEnabled`. `CarouselState` exposes
`currentItem` and programmatic scrolling.

### Pull-to-refresh migration (material3-1.4.0)

In `PullToRefreshDefaults`, rename `shape` to `indicatorShape` and
`containerColor` to `indicatorContainerColor`. `indicatorMaxDistance` is new.
Custom `PullToRefreshState` implementations must implement `isAnimating`;
there is no inherited default implementation.

### Slider state and layouts (material3-1.4.0)

Hoist state with `rememberSliderState` and `rememberRangeSliderState`.
`SliderState.shouldAutoSnap` can turn off automatic snapping for a custom
animation, and `onValueChange` is public. Additional layouts include
`VerticalSlider`, center-origin `CenteredTrack`, external track-corner and icon
customization, and `trackCornerSize` for range-slider tracks.

## Tooltips and sheets

### Tooltip positioning and dismissal (material3-1.4.0)

Use `rememberTooltipPositionProvider` instead of deprecated plain and rich
position-provider functions. `TooltipScope.layoutCoordinates` exposes the
anchor and supersedes `drawCaret`; tooltips can define custom caret shapes and
positions.

The dismissal overload takes `onDismissRequest`, and `onDismiss` is no longer
suspending. `TooltipBox` defaults `focusable` to `false` and adds `hasAction`.
Plain and rich tooltips default to maximum widths of 200 dp and 320 dp.

### Bottom-sheet controls (material3-1.4.0)

`ModalBottomSheet` adds `sheetGestureEnabled`. `ModalBottomSheetProperties` can
prevent scrim clicks from requesting dismissal; light status- and
navigation-bar options are Android-only. `SheetState.isAnimationRunning` is
public. Its density constructor is deprecated in favor of positional and
velocity thresholds. `BottomSheetDefaults.windowInsets` includes
`WindowInsets.safeDrawing.Top`.

## Insets

### Display-cutout defaults (material3-1.4.0)

Inset-aware Material 2 and Material 3 components include `displayCutout` in
their default `WindowInsets`. Override the component's inset parameter when the
layout intentionally draws into the cutout area.

## Material migration checklist

- Inspect resolved icon dependencies after upgrading Material 3.
- Recheck selected navigation label colors against the design system.
- Keep stable and expressive-alpha API expectations separate.
- Hoist state for text fields, search, carousels, and sliders.
- Test expanded search, dialogs, tooltips, and sheets as separate windows where
  applicable.
- Recheck default insets on devices with display cutouts.
