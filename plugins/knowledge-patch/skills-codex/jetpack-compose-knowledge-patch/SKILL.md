---
name: jetpack-compose-knowledge-patch
description: Jetpack Compose
version: null
license: MIT
metadata:
  author: Nevaberry
---





# Jetpack Compose Knowledge Patch

Use this skill when upgrading, reviewing, or debugging modern Jetpack Compose
code across Compose Runtime, UI, Foundation, Animation, Material 3, compiler,
Gradle, Android hosts, and Compose UI tests.

## How to use this patch

1. Identify the affected artifact and its actual resolved version. A Compose BOM
   aligns library versions but does not make every library share one version.
2. Check the breaking-change guidance first, especially toolchain floors,
   removed flags, renamed APIs, state ownership, and test scheduling.
3. Open the topic reference that matches the code being changed.
4. Prefer the project's manifests, source, and tests when its dependency set
   differs from the APIs described here.
5. Verify Android-only behavior separately from common or multiplatform APIs.

## Reference index

| Reference | Topics |
| --- | --- |
| [Android hosts, windows, and insets](references/android-host-windows.md) | Compose views, resources, host defaults, windows, insets, haptics, clipboard, Android interop |
| [Build, runtime, and state](references/build-runtime-state.md) | SDK and Gradle floors, compiler, BOMs, snapshots, saveable and retained state, runtime diagnostics |
| [Input, focus, and scrolling](references/input-focus-scrolling.md) | Focus, gestures, overscroll, pointer input, lazy layouts, scroll state, visibility |
| [Layout, animation, and graphics](references/layout-animation-graphics.md) | Lookahead, transitions, FlexBox, Grid, modifier nodes, shadows, shaders, layers |
| [Material 3 components](references/material3-components.md) | Material migrations, text fields, search, navigation, pickers, sheets, sliders, tooltips |
| [Semantics, accessibility, and testing](references/semantics-accessibility-testing.md) | Autofill, semantics trees, accessibility, rules, dispatchers, restoration |
| [Text and editing](references/text-editing.md) | Autosizing, overflow, annotations, undo, transformations, menus, secure fields, IME state |

## Breaking changes first

### Check the Android and Kotlin floors

- Compose Animation, Foundation, Runtime, and UI require Android API 23 or
  newer. Do not assume the historical API 21 floor still applies.
- Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or newer.
- Compose lint requires AGP 8.8.2 and Android Studio Ladybug or newer. An older
  AGP can select standalone Lint 8.8.2 in `gradle.properties`.
- Android projects adopting Compose 1.12 artifacts must compile against SDK 37
  with AGP 9. `compileSdk` does not force the same `targetSdk`.

See [Build, runtime, and state](references/build-runtime-state.md) before
changing the build.

### Finish the indication migration

Recompiled interaction modifiers obtain an `IndicationNodeFactory` from
`LocalIndication`. Supplying the old `Indication` can fail at runtime. Migrate
the indication or use an overload with an explicit indication during a staged
upgrade. The temporary non-composed-clickable escape flag is no longer
available in newer Foundation versions.

### Delete removed behavior-flag assignments

Several temporary Foundation, UI, and Runtime flags were removed after their
behaviors became unconditional or their migrations completed. Treat an
unresolved flag as migration work, not as a symbol to recreate. The complete
list and replacement behavior are in
[Input, focus, and scrolling](references/input-focus-scrolling.md).

### Migrate saveable and retained state deliberately

- Remove custom `key` arguments from `rememberSaveable`; positional scoping is
  required for reliable state ownership, especially inside lazy layouts.
- Use `rememberSerializable` for the `KSerializer` overload. The `Saver`
  overload remains `rememberSaveable`.
- Use `retain` only for non-serialized values that may survive leaving the
  composition. Avoid resource-owning keys and mark unsuitable types with
  `@DoNotRetain`.
- Install custom retained-value stores with
  `LocalRetainedValuesStoreProvider`; do not directly provide the store local.

### Account for dispatcher changes in UI tests

The v2 Compose UI test APIs use `StandardTestDispatcher` by default. Work stays
queued until the shared scheduler advances, so call `runCurrent()` when the
test expects due coroutine work. Older test APIs retain unconfined scheduling.

### Update Material 3 dependencies and removed APIs

- Material 3 no longer brings in `material-icons-core`. Declare it explicitly
  only for existing icons; prefer Material Symbols vector resources for new UI.
- Stable Material 3 removed APIs still carrying expressive or component-
  override experimental annotations; those APIs belong to a different alpha
  line.
- Replace `TabRow` and `ScrollableTabRow` with the appropriate primary or
  secondary variants.
- Supply both fixed-color and surface-container role families when constructing
  a custom `ColorScheme`.

## High-value current APIs

### Observe geometry and visibility efficiently

Use `Modifier.onLayoutRectChanged` for debounced or throttled bounds in root,
window, or screen coordinates. Use `onVisibilityChanged` for visibility state;
`onFirstVisible` is deprecated because it can fire again after an item leaves
and re-enters the viewport.

### Build state-backed text input

Use `TextFieldState` with `OutputTransformation`. Rendered styling belongs in
`TextFieldBuffer.addStyle`; programmatic
`TextFieldState.edit` creates an undo entry unless history is explicitly
cleared. Material 3 supplies state-backed normal, outlined, and secure fields.

### Separate scrolling, clipping, and indicators

`Modifier.scrollableArea()` combines scroll handling with bounds clipping.
`ScrollableState.scrollIndicatorState` exposes indicator data, while
`Modifier.scrollIndicator` and `ScrollIndicatorFactory` render custom
indicators. For custom overscroll, keep event handling and drawing in the
intended owners and never draw one effect twice.

### Choose the right two-dimensional layout

- `FlexBox` covers row, column, and wrapping flex layouts with grow and shrink.
- Experimental `Grid` provides explicit CSS-like tracks and placement without
  lazy composition.
- Stable `LazyLayout` primitives support custom virtualized layouts and
  internal prefetch scheduling.
- Use ordinary `FlowRow` or `FlowColumn` without the deprecated overflow
  overloads for simpler wrapping layouts.

### Use retained values for non-serialized continuity

`retain` bridges a gap between `remember` and saveable state: values can outlive
their current composition placement without being serialized. On Android, a
lifecycle-aware retain scope can span configuration changes. Pair lifecycle
work with `RetainedEffect`, not a composition-only effect.

### Diagnose animations and minified stack traces

Lookahead visual-debugging APIs show target bounds, paths, matches, and active
shared transitions. For runtime failures, group-key stack traces can work in
minified apps when mapping generation is enabled by a compatible compiler
plugin. Compose diagnostic stack traces also cover `LaunchedEffect` and
`rememberCoroutineScope` work.

## Common migration patterns

### Focus

Use receiver-based `FocusProperties.onEnter` and `onExit`, directional
`requestFocus`, and `focusRestorer(fallback = ...)`. Mouse or touchpad presses
outside the focused node clear focus by default; disable that behavior per view
with `AbstractComposeView.isClearFocusOnPointerDownEnabled = false`.

### Insets

Compose views pass window insets through by default. Use
`recalculateWindowInsets()` after ancestor alignment when descendants need
correct padding, and use `disableWindowInsetsRulers()` per Compose view when
rulers must be disabled.

### Semantics and autofill

Use typed `fillableData` and `onFillData` rather than the old text-only autofill
action. Replace `invisibleToUser()` with `hideFromAccessibility()`, fetch a
semantics node to read its ID, and avoid tests that assume decoration modifiers
never insert semantics nodes.

### Graphics

Add `graphicsLayer()` explicitly when code depended on `BasicText` creating an
implicit layer. Convert packed Compose colors before comparing them with
Android color longs, and use `Paint.nativePaint` instead of the deprecated
platform paint typealias bridge.

## Validation checklist

- Resolve the actual artifact versions after BOM alignment.
- Compile with the required SDK, AGP, Kotlin, and lint versions.
- Search for deprecated names and removed compatibility flags.
- Exercise focus, pointer, nested scrolling, and accessibility behavior on a
  real Android host when those behaviors matter.
- Advance the test coroutine scheduler explicitly where v2 test APIs are used.
- Re-run semantics tests with matchers resilient to inserted semantics nodes.
- Check window insets and cutouts on edge-to-edge and overlay windows.
- Confirm that custom retained stores, effects, and resource-owning values have
  the intended lifecycle.
