---
name: jetpack-compose-knowledge-patch
description: Jetpack Compose
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Jetpack Compose

Use this skill when upgrading or maintaining AndroidX Compose UI, Foundation,
Runtime, Animation, testing, or Material 3 code. Check the project's Compose
BOM and individually pinned artifacts before applying guidance because the
libraries release independently.

## How to use this patch

1. Inspect the module's Compose BOM, direct artifact versions, Kotlin plugin,
   Android Gradle Plugin, `compileSdk`, and `minSdk`.
2. Open the topic reference that matches the code being changed.
3. Apply advice only to artifacts at or beyond the version cited beside it.
4. Prefer current APIs over temporary compatibility flags; several flags were
   removed in later Foundation and UI releases.
5. Run Compose UI tests with the dispatcher and accessibility artifacts that
   match the selected testing API generation.

## Reference index

| Reference | Topics |
| --- | --- |
| [Setup, Runtime, and State](references/setup-runtime-state.md) | Toolchain floors, BOMs, runtime annotations, saveable state, retention, composition, diagnostics |
| [Input, Scrolling, and Focus](references/input-scroll-focus.md) | Focus, gestures, overscroll, 2D scrolling, lazy layout, visibility, pointer input |
| [Layout, Animation, and Graphics](references/layout-animation-graphics.md) | Lookahead, shared transitions, layout nodes, FlexBox, Grid, shadows, shaders, frame rate |
| [Material 3 Components](references/material3-components.md) | Material 3 dependencies, navigation, text fields, search, pickers, tooltips, sheets, sliders, color |
| [Platform, Window, and Interop](references/platform-window-interop.md) | Insets, window geometry, Android hosts, resources, clipboard, paint, platform interop |
| [Semantics, Accessibility, and Testing](references/semantics-accessibility-testing.md) | Semantics migrations, accessibility checks, test dispatchers, test rules, host theme |
| [Text, Autofill, and Resources](references/text-autofill-resources.md) | Semantic autofill, text sizing and overflow, editing, context menus, selection, transliteration |

## Upgrade blockers and removed APIs

### Align the Android toolchain

- Starting with Compose 1.12, use `compileSdk = 37` and Android Gradle Plugin
  9. This does not require `targetSdk = 37`.
- Compose Animation, Foundation, Runtime, and UI 1.10.0 require API 23 or
  newer. Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or
  newer.
- Compose lint 1.9.0 requires AGP 8.8.2 and Android Studio Ladybug or newer.
  On an older AGP, select standalone Lint 8.8.2 with
  `android.experimental.lint.version=8.8.2`.

### Finish the indication migration

After recompiling against Foundation 1.9.0, indication-less `clickable`,
`combinedClickable`, `selectable`, `toggleable`, and `triStateToggleable`
overloads require an `IndicationNodeFactory` from `LocalIndication`. A legacy
`Indication` can cause a runtime crash. Migrate it or temporarily use an
explicit-indication overload. The compatibility flag was removed in 1.10.0.

### Remove obsolete behavior flags

Delete assignments to behavior flags that disappeared when their behavior
became permanent. This includes the Foundation flags for non-composed
clickables, on-scroll callbacks, fling continuation, drag pickup, nested
prefetch, pointer-velocity and interop fixes, immediate tap dispatch,
non-suspending clickable input, text-field D-pad navigation, and keep-in-view
focus observation. Also remove the UI semantic-autofill flag and the Runtime
movable-content flags.

### Migrate renamed and removed APIs

- Replace `FocusProperties.enter` and `exit` with `onEnter` and `onExit`.
- Pass `fallback` to `Modifier.focusRestorer()` instead of a lambda.
- Replace `AutoSize` with `TextAutoSize`.
- Replace `invisibleToUser()` with `hideFromAccessibility()`.
- Replace `Snapshot.id` with `snapshotId`.
- Replace `currentCompositeKeyHash` with `currentCompositeKeyHashCode`.
- Replace `TabRow` and `ScrollableTabRow` with primary or secondary variants.
- Replace `Modifier.onFirstVisible()` with `onVisibilityChanged()` plus
  explicit prior-visibility tracking.
- Replace `RetainedValuesStore.getExitedValueOrDefault` with
  `consumeExitedValueOrDefault`.
- Use `android.graphics.Paint` and `Paint.nativePaint`, not `NativePaint` and
  `asFrameworkPaint()`.

## Runtime, state, and dependency quick reference

### Use the BOM consistently

Import the same Compose platform in application and instrumented-test
configurations, then omit versions from individual Compose dependencies:

```kotlin
val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
implementation(composeBom)
androidTestImplementation(composeBom)
implementation("androidx.compose.material3:material3")
```

Use `compose-bom-alpha` for the newest alpha-or-better artifacts or
`compose-bom-beta` for beta-or-better artifacts. These testing BOMs may also
resolve some libraries to stable versions.

### Choose the right state lifetime

- Use `rememberSaveable` for serializable state that must survive recreation.
  Do not pass its deprecated custom `key`; positional scoping prevents state
  sharing and loss in nested lazy layouts.
- Use `rememberSerializable` for the `KSerializer` overload.
- Use `retain` for nonserialized values that should outlive a temporary exit
  from the composition. Avoid resource-leaking keys and mark unsuitable types
  with `@DoNotRetain`.
- Install custom retention stores with `LocalRetainedValuesStoreProvider` and
  use `ManagedRetainedValuesStore` rather than directly providing the local.

### Treat pausable and retained lifecycles explicitly

Pausable composition requires compiler support. Check `isApplied` and
`isCancelled`; dispose a cancelled pausable composition because reuse throws.
Use `RetainedEffect` for retention lifetime work and `onUnused` for values
abandoned before use.

## Input, scrolling, and focus quick reference

### Configure semantic autofill

Text autofill in Compose 1.8.0 requires both UI and Foundation 1.8 or newer.
Use semantic autofill APIs and `FillableData.createFrom(value)`. In 1.10.0,
use `fillableData` and `onFillData` rather than `onAutofillText`; date values
are available through `dateMillisValue`.

### Own overscroll exactly once

Use `rememberPlatformOverscrollFactory` and `LocalOverscrollFactory`. Provide
`null` to disable overscroll or create a factory with `color` and `padding` to
customize it. A custom `OverscrollEffect` may split event handling and drawing
with `withoutVisualEffect` and `withoutEventHandling`; never draw one effect
twice.

### Handle modern scrolling

- Use `Modifier.scrollable2D` and `Scrollable2DState` for two-axis input; the
  `canScroll` contract takes an `Offset`.
- Use `Modifier.scrollableArea()` when scrolling also needs bounds clipping
  and automatic content-direction handling.
- Read scrollbar state from `ScrollableState.scrollIndicatorState`, or build
  a custom indicator with `Modifier.scrollIndicator` and
  `ScrollIndicatorFactory`.
- Direct composition reads of `ScrollState.value` and
  `PagerState.currentPageOffsetFraction` are frequently invalidating.

## Layout, animation, and graphics quick reference

### Animate with lookahead and shared transitions

Use `Modifier.animateBounds` inside a lookahead scope. Shared-transition APIs
support dynamic enablement, disposed-target fallback bounds, initial gesture
velocity, lookahead coordinates, and `skipToLookaheadPosition`. Use
`scaleToBounds`; removed `ScaleToBounds` and legacy configuration parameters
must not remain in upgraded code.

### Select a general or explicit layout

Use `FlexBox` with `FlexBoxConfig` and `Modifier.flex` for grow, shrink,
wrapping, direction, and alignment. Its DSL uses `grow(1f)` calls. Content
that cannot shrink may overflow, so add `clipToBounds()` when needed.

Use experimental `Grid` for explicit two-dimensional tracks and placement.
When a flexible track contains a `SubcomposeLayout`,
`MinMax(0.dp, 1.fr)` avoids intrinsic queries.

## Material 3 quick reference

### Declare icons intentionally

Material 3 1.4.0 no longer brings in `material-icons-core`. Declare it only
for existing icon usage; for new work prefer Material Symbols Vector Drawable
XML.

### Use state-backed components

- Prefer `TextFieldState` overloads for text fields and the stable decoration
  APIs; use `SecureTextField` or `OutlinedSecureTextField` for passwords.
- Drive collapsed and expanded search surfaces with `SearchBarState`; use
  `ExpandedFullScreenSearchBar` or `ExpandedDockedSearchBar` for the expanded
  window.
- Hoist sliders with `rememberSliderState` or `rememberRangeSliderState`.
- Drive carousels through `CarouselState`, including `currentItem` and
  programmatic scrolling.

### Account for changed defaults

- Selected navigation item labels use `colorScheme.secondary`.
- `TooltipBox` defaults `focusable` to `false`; plain and rich tooltip maximum
  widths are 200 dp and 320 dp.
- Default component insets include display cutouts, and bottom sheets use
  `WindowInsets.safeDrawing.Top`.
- Custom `ColorScheme` construction should supply fixed roles and
  surface-container roles.

## Testing quick reference

Accessibility checks live in `ui-test-accessibility` without a rule and
`ui-test-junit4-accessibility` when invoked on a JUnit 4 rule. Compose UI
testing v2 uses `StandardTestDispatcher` by default, exposes the shared
`TestCoroutineScheduler`, and requires explicit scheduling such as
`runCurrent()` for queued coroutines. Deprecated test APIs retain
`UnconfinedTestDispatcher` behavior.
