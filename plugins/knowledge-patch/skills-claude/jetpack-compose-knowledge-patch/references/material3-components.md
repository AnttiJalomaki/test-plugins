# Material 3 Components

## Dependencies, theme, and defaults

### Material Icons removal (material3-1.4.0)

Material 3 no longer adds `material-icons-core` transitively. Declare it
explicitly if existing code still uses it. The
`androidx.compose.material.icons` library is no longer updated or recommended;
for new icons, prefer Material Symbols Vector Drawable XML downloaded from the
Android tab of the Material icons site.

### Navigation-item label color (material3-1.4.0)

Selected labels in `NavigationBarItem` and `NavigationRailItem` use
`MaterialTheme.colorScheme.secondary` instead of `onSurface`. To preserve the
old look, copy the default colors and set `selectedTextColor` to
`MaterialTheme.colorScheme.onSurface`.

### Motion schemes (material3-1.4.0)

Components take motion from a `MotionScheme` supplied through `MaterialTheme`.
Modifier nodes can read it with
`currentValueOf(MotionTheme.LocalMotionScheme)`. Create the standard scheme
with `MotionScheme.standard()`.

### Expressive and override APIs (material3-1.4.0)

The stable line removes all public APIs still marked
`ExperimentalMaterial3ExpressiveApi` or
`ExperimentalMaterial3ComponentOverrideApi`. Do not retain calls to those APIs
when staying on Material 3 1.4.

### Display-cutout insets (material3-1.4.0)

Inset-aware Material 2 and Material 3 components include `displayCutout` in
their default `WindowInsets`. Override the relevant component inset parameter
if content should not avoid the cutout.

## Text fields

### State-backed and secure fields (material3-1.4.0)

`TextField` and `OutlinedTextField` have `TextFieldState` overloads. Their
`TextFieldDecorator`-compatible decoration APIs are stable. Use
`labelPosition` to keep a label minimized so an unfocused placeholder remains
visible.

Use `SecureTextField` or `OutlinedSecureTextField` for password entry.

## Search

### State and expanded views (material3-1.4.0)

Search UI is split between a collapsed `SearchBar` and expanded
`ExpandedFullScreenSearchBar` or `ExpandedDockedSearchBar` surfaces, which
open in a new window. Drive them with `SearchBarState`.

`TopSearchBar` adds inset and scroll handling. `InputField` has a state-backed
overload.

## Pickers

### Time picker (material3-1.4.0)

`TimePickerDialog` can host `TimePicker`, `TimeInput`, or a UI that switches
between them. Replace `TimePickerState.isAfternoon` with the `isPm` extension
property.

### Date and range pickers (material3-1.4.0)

`DatePicker`, `DateRangePicker`, and their supporting APIs are stable. State
factories and state extensions support `LocalDate` and `YearMonth` on API 26+
or with desugaring. `getDisplayedMonth()` is non-null.

The input-mode focus option accepts an optional `FocusRequester`, not a
Boolean. Directly changing the state locale does not localize the default
title or headline, so supply localized text explicitly when required.

## Carousels

### Hero layout and state (material3-1.4.0)

`HorizontalCenteredHeroCarousel` provides a center-aligned hero layout.
Carousel composables accept `userScrollEnabled`. `CarouselState` exposes
`currentItem` and programmatic scrolling.

## Pull to refresh

### Defaults and custom state (material3-1.4.0)

In `PullToRefreshDefaults`, rename `shape` to `indicatorShape` and
`containerColor` to `indicatorContainerColor`. `indicatorMaxDistance` is new.
Custom `PullToRefreshState` implementations must implement `isAnimating`;
there is no longer a default implementation to inherit.

## Tooltips

### Positioning, caret, and dismissal (material3-1.4.0)

Use `rememberTooltipPositionProvider` instead of the deprecated plain and rich
position-provider functions. `TooltipScope.layoutCoordinates` exposes the
anchor and supersedes `drawCaret`; tooltips support custom caret shapes and
positions.

The new dismissal overload accepts `onDismissRequest`, and `onDismiss` is no
longer suspend. `TooltipBox` defaults `focusable` to `false` and adds
`hasAction`. Plain and rich tooltips default to maximum widths of 200 dp and
320 dp, respectively.

## Bottom sheets

### Gestures, dismissal, and insets (material3-1.4.0)

`ModalBottomSheet` adds `sheetGestureEnabled`.
`ModalBottomSheetProperties` can prevent scrim clicks from requesting
dismissal; its light-status-bar and light-navigation-bar options are
Android-only.

`SheetState.isAnimationRunning` is public. Its density constructor is
deprecated in favor of positional and velocity thresholds.
`BottomSheetDefaults.windowInsets` includes `WindowInsets.safeDrawing.Top`.

## Navigation

### Tab rows (1.9.0)

`TabRow` and `ScrollableTabRow` are deprecated. Migrate to the appropriate
primary or secondary tab-row variant.

### Wide rails and short bars (material3-1.4.0)

`WideNavigationRail`, `ShortNavigationBar`, and `NavigationItem` are stable.
`WideNavigationRailItem` requires `railExpanded`, and
`WideNavigationRailState` exposes Boolean current and target values.

Use `Arrangement.Vertical` instead of `WideNavigationRailArrangement`. Use
the renamed shape defaults under `WideNavigationRailDefaults` because
`ModalWideNavigationRailDefaults` was removed.

### Navigation suites (material3-1.4.0)

`NavigationSuite`, `NavigationSuiteItem`, `NavigationSuiteColors`, and
`NavigationSuiteTypes` support additional layouts selected with
`navigationSuiteType`. Corresponding scaffold and layout APIs accept optional
primary-action content.

## Sliders

### Hoisted state and layouts (material3-1.4.0)

Use `rememberSliderState` and `rememberRangeSliderState`. Set
`SliderState.shouldAutoSnap` to `false` for custom snapping animation;
`onValueChange` is public.

Material 3 also provides `VerticalSlider`, center-origin `CenteredTrack`,
custom external track corners and icons, and `trackCornerSize` for range
slider tracks.

## Color schemes and annotated links

### Constructor requirements (material3-1.4.0)

The `ColorScheme` constructor without fixed color roles is deprecated. The
constructor without surface-container roles is hidden. Custom schemes should
supply both role families.

`ColorScheme.contentColorFor(surfaceDim)` resolves to `onSurface`. Links in
`Text(AnnotatedString)` receive Material styling by default.
