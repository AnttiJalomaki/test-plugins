---
name: jetpack-compose-knowledge-patch
description: Jetpack Compose
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Jetpack Compose

Use this skill when upgrading or maintaining Jetpack Compose UI, Foundation,
Runtime, Animation, testing, compiler, BOM, or Material 3 code. Start with the
project's actual dependency graph and toolchain, then apply only the guidance
that matches the affected artifact.

## How to use this skill

1. Inspect the version catalog, Gradle dependencies, Compose BOM, Kotlin
   version, AGP version, `compileSdk`, and explicit artifact overrides.
2. Identify which Compose products are changing. Compose UI and Material 3 do
   not share one release train.
3. Read the indexed reference for the affected API area.
4. Search for removed APIs, deprecated overloads, temporary flags, and behavior
   assumptions before changing dependencies.
5. Compile all source sets and run UI, accessibility, state-restoration, and
   screenshot tests appropriate to the change.
6. Trust the project's code, resolved dependencies, compiler diagnostics, and
   tests when they reveal a more specific constraint.

## Reference index

| Reference | Topics |
|---|---|
| [setup-runtime-state.md](references/setup-runtime-state.md) | BOMs, Android and Kotlin floors, compiler configuration, composition, snapshots, saving, retention, runtime diagnostics |
| [layout-animation-graphics.md](references/layout-animation-graphics.md) | Lookahead, shared transitions, layout primitives, modifier nodes, visibility, shadows, shaders, frame rates |
| [input-scroll-focus.md](references/input-scroll-focus.md) | Scrolling, overscroll, gestures, pointer input, focus, indications, haptics, selection |
| [text-autofill-resources.md](references/text-autofill-resources.md) | Text fields, autofill, autosizing, annotations, fonts, clipboard, resources |
| [semantics-accessibility-testing.md](references/semantics-accessibility-testing.md) | Semantics, accessibility, UI tests, scheduling, diagnostics |
| [material3-components.md](references/material3-components.md) | Material 3 dependency changes, themes, navigation, search, pickers, sheets, sliders, tooltips |
| [platform-window-interop.md](references/platform-window-interop.md) | Insets, window geometry, Android hosts, dialogs, popups, paint, packed colors, multiplatform artifacts |

## Breaking changes first

### Check the Android build floor

Compose 1.12 Android projects require AGP 9 and `compileSdk = 37`.
`compileSdk` does not force the same `targetSdk`.

```kotlin
android {
    compileSdk = 37
}
```

Compose Animation, Foundation, Runtime, and UI 1.10 raise `minSdk` to 23.
Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or newer.
Compose lint from 1.9 requires AGP 8.8.2 or standalone Lint 8.8.2 or newer.

### Treat BOM choice as a release-channel choice

Use the ordinary BOM for its mapped stable set. Use `compose-bom-alpha` to
select each library's latest alpha-or-better release, or `compose-bom-beta` for
its latest beta-or-better release. Either prerelease BOM may still map some
libraries to stable versions.

Import the chosen platform in both application and instrumented-test
configurations and omit individual Compose library versions.

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    implementation("androidx.compose.material3:material3")
}
```

### Finish the indication migration

After recompiling against Foundation 1.9, indication-less `clickable`,
`combinedClickable`, `selectable`, `toggleable`, and `triStateToggleable`
overloads require an `IndicationNodeFactory` from `LocalIndication`. A legacy
`Indication` can fail at runtime. Migrate it or temporarily use an overload with
an explicit indication. The compatibility flag is gone in Foundation 1.10.

### Remove expired behavior flags

Do not carry old `ComposeFoundationFlags`, Compose UI flags, or runtime flags
forward blindly. Several scroll, gesture, pointer, prefetch, autofill,
text-field navigation, keep-in-view, and movable-content behaviors became
unconditional before their flags were removed. See the input and runtime
references for the exact names.

### Migrate saveable and retained state deliberately

Remove custom keys from `rememberSaveable`; positional scoping is the supported
model. Use `rememberSerializable` for `KSerializer` and keep `rememberSaveable`
for `Saver`. Import `LocalSavedStateRegistryOwner` from
`androidx.savedstate.compose`.

Use `retain` for non-serialized values that may outlive composition but do not
need saveable-state lifetime. Avoid resource-leaking keys and mark unsuitable
types with `@DoNotRetain`. Install custom retained stores through
`LocalRetainedValuesStoreProvider`.

### Account for semantics-tree changes

`background`, `border`, and `graphicsLayer` can introduce semantics nodes.
Tests that assume exact parent, sibling, or child structure may break. Tag the
target node directly or use a looser ancestor matcher.

Replace `invisibleToUser()` with `hideFromAccessibility()`. Retrieve a
semantics ID through `fetchSemanticsNode().id`; `semanticsId()` was removed.

### Update test scheduling assumptions

Compose UI testing v2 uses `StandardTestDispatcher` by default. Coroutines stay
queued until the shared scheduler advances, so call `runCurrent()` where
needed. Older test variants retain `UnconfinedTestDispatcher`.

The stable `effectContext` rule factories also accept
`StandardTestDispatcher`; call `MainTestClock.runCurrent()` to execute due
work.

### Keep Material artifacts explicit

Material 3 no longer brings in `material-icons-core`. Declare it explicitly
only while migrating; prefer downloaded Material Symbols vector XML.

For Material 3 prerelease ripple use cases, prefer Material 3's built-in themed
ripple. Add `androidx.compose.material3:material3-ripple` directly only when
replacing direct `material-ripple` use or when inset focus rings are needed
without the full Material 3 library.

## High-value API guidance

### Observe geometry and visibility cheaply

Prefer `Modifier.onLayoutRectChanged` over `onGloballyPositioned` when bounds
observation needs debounce, throttle, or root/window/screen coordinates.
Use `LocalWindowInfo.current.containerSize` for the current content container.

Use `onVisibilityChanged` for repeatable visibility state. `onFirstVisible` is
deprecated because it can fire after every re-entry; track the first transition
yourself when that distinction matters.

### Use the new layout primitives for the right job

`Modifier.animateBounds` animates lookahead size and position changes. Lazy
grids and pagers participate in lookahead.

`FlexBox` handles grow, shrink, wrapping, direction, and alignment. Its DSL
uses function calls such as `grow(1f)`. Add `clipToBounds()` when unshrinkable
content must not overflow.

Experimental `Grid` provides explicit two-dimensional tracks and
`Modifier.gridItem()`. Use `MinMax(0.dp, 1.fr)` around subcomposed content to
avoid intrinsic measurement.

### Separate overscroll input from drawing

Configure platform overscroll with `rememberPlatformOverscrollFactory` and
`LocalOverscrollFactory`; provide `null` to disable it. A custom
`OverscrollEffect` can split event handling from rendering through
`withoutVisualEffect` and `withoutEventHandling`. Ensure the effect is drawn
exactly once.

### Prefer state-backed text APIs

State-backed text fields support styled output through
`OutputTransformation` and `TextFieldBuffer.addStyle`. A programmatic
`TextFieldState.edit {}` creates its own undo entry; call
`undoState.clearHistory()` only when reset behavior is intentional.

Use `TextAutoSize`, not removed `AutoSize` overloads. Start and middle ellipsis
require a single line.

### Use typed semantic autofill

Modern autofill is semantics based. Text autofill requires matching UI and
Foundation support. Use `fillableData` and `onFillData`; construct typed values
with `FillableData.createFrom(value)`. Do not return to deprecated manager or
`onAutofillText` APIs.

### Choose a search-bar state model

Use `SearchBarState` and slot-based Material 3 search APIs. The older
`expanded`/`onExpandedChange` overloads are deprecated. Collapsed `SearchBar`
and expanded full-screen or docked views are separate surfaces.

### Request and render scroll indicators explicitly

Read scrollbar information through `ScrollableState.scrollIndicatorState`.
Use `Modifier.scrollIndicator` and a `ScrollIndicatorFactory` for a custom
indicator. Material 3 scrollbar APIs require a non-null indicator state.

### Debug transitions and recomposition with current hooks

Lookahead visual-debugging APIs can draw target bounds, trajectories,
shared-element matches, and active-transition state. Runtime diagnostics can
enable group-key stack traces, inspect recomposer error state, and emit
compiler reports configured through the module-level `composeCompiler` block.

## Upgrade checklist

- Resolve the dependency graph and record which artifacts come from a BOM.
- Confirm AGP, Kotlin, `compileSdk`, `minSdk`, and lint meet the selected
  artifacts' requirements.
- Compile for removed overloads, renamed APIs, experimental opt-ins, and
  non-null parameter changes.
- Search for assignments to obsolete Compose flags.
- Audit custom indications, modifier nodes, retained stores, saveable keys,
  window-inset consumption, and Material icon dependencies.
- Re-run semantics and accessibility tests without assuming exact tree shape.
- Advance the test scheduler explicitly when using v2 APIs.
- Exercise keyboard, mouse, trackpad, touch, nested scrolling, and focus paths
  used by the app.
- Test configuration changes, state restoration, dialogs, popups, and detached
  `ComposeView` hosts where applicable.
- Verify prerelease Material 3 APIs and opt-ins independently of core Compose.
