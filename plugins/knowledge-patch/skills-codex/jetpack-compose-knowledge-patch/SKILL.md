---
name: jetpack-compose-knowledge-patch
description: Jetpack Compose
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Jetpack Compose Knowledge Patch

Use this skill when upgrading, configuring, debugging, or testing AndroidX
Compose UI, Foundation, Runtime, Animation, or Material 3. It focuses on API
migrations, changed defaults, toolchain constraints, and newer capabilities
that commonly invalidate older Compose code.

Treat the project's dependency graph as authoritative. Compose libraries do
not all advance together, and a BOM can select different stable or prerelease
versions for different artifacts. Check the resolved artifact containing an
API before applying version-specific advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [build-runtime-state.md](references/build-runtime-state.md) | Android and Kotlin floors, compiler configuration, BOMs, runtime annotations, snapshots, saveable state, pausing, retention, diagnostics |
| [input-focus-scrolling.md](references/input-focus-scrolling.md) | Autofill, focus, haptics, gestures, overscroll, scrolling, visibility, trackpad input |
| [layout-animation-graphics.md](references/layout-animation-graphics.md) | Lookahead, shared transitions, FlexBox, Grid, lazy layouts, modifier nodes, drawing, shaders, shadows |
| [text-editing.md](references/text-editing.md) | Autosizing, ellipsis, annotations, state-backed editing, context menus, selection, secure fields, fonts |
| [semantics-accessibility-testing.md](references/semantics-accessibility-testing.md) | Semantics changes, accessibility behavior, test artifacts, dispatcher scheduling, restoration |
| [android-host-windows.md](references/android-host-windows.md) | Insets, window geometry, resources, host composition, dialogs, popups, Android interop |
| [material3-components.md](references/material3-components.md) | Material 3 dependencies, search, pickers, navigation, sheets, tooltips, sliders, color and ripple migrations |

## Breaking changes and migrations

### Resolve build constraints first

- Compose Animation, Foundation, Runtime, and UI require Android API 23 or
  newer when using artifacts with the raised minimum SDK.
- Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or newer.
- Compose lint checks need AGP 8.8.2 or a standalone Lint 8.8.2 override.
- Projects moving to Compose 1.12 must compile with SDK 37 and AGP 9. This
  does not force `targetSdk` 37.
- Import the same Compose BOM into application and instrumented-test
  configurations, then omit versions from individual Compose dependencies.

### Remove expired compatibility flags

- Do not rely on `ComposeFoundationFlags.isNonComposedClickableEnabled`; the
  temporary bridge for old `Indication` implementations was removed. Supply
  an `IndicationNodeFactory` through `LocalIndication` or use an overload with
  an explicit indication while migrating.
- Delete assignments to removed flags for immediate tap dispatch, scroll
  callbacks, fling continuation, drag pickup, nested prefetch, pointer
  interop, semantic autofill, text-field D-pad navigation, keep-in-view focus
  observation, and movable-content behavior.
- Delete `ComposeUiTestFlags.isStandardTestDispatcherSupportEnabled`; testing
  v2 selects `StandardTestDispatcher` directly.

### Replace renamed and removed APIs

- Replace `AutoSize` with `TextAutoSize`.
- Replace `FocusProperties.enter` and `exit` with receiver-based `onEnter` and
  `onExit`. Pass a non-null `fallback` requester to `focusRestorer`.
- Replace `OverscrollConfiguration` and `LocalOverscrollConfiguration` with
  `rememberPlatformOverscrollFactory` and `LocalOverscrollFactory`.
- Replace `invisibleToUser()` with `hideFromAccessibility()` and obtain a
  semantics ID from `fetchSemanticsNode().id`.
- Replace `Snapshot.id` with `Snapshot.snapshotId`, and
  `currentCompositeKeyHash` with `currentCompositeKeyHashCode`.
- Remove the custom `key` argument from `rememberSaveable`; positional scoping
  prevents state sharing and loss in nested lazy layouts.
- Replace `ScaleToBounds` with `scaleToBounds`; removed shared-transition
  factories and parameters have no direct compatibility shim.
- Replace indirect-touch API names with their indirect-pointer equivalents.
- Replace `UnplacedStateAwareModifierNode` with `UnplacedAwareModifierNode`
  and `invalidateLayoutForSubtree` with `invalidateMeasurementForSubtree`.
- Replace `NativePaint` with `android.graphics.Paint` and
  `asFrameworkPaint()` with the `nativePaint` extension.

### Account for changed behavior

- `AbstractComposeView.consumeWindowInsets` defaults to `false`; set it to
  `true` only to retain consuming behavior.
- `TextFieldState.edit {}` creates an undo entry. Explicitly call
  `undoState.clearHistory()` when a programmatic edit should reset history.
- `background`, `border`, and `graphicsLayer` can add semantics nodes. Avoid
  tests that depend on exact parent, child, or sibling structure.
- `BasicText` no longer creates an implicit graphics layer. Add
  `Modifier.graphicsLayer()` only when that layer's behavior is required.
- Pointer presses outside a focused node clear focus by default; opt out per
  `AbstractComposeView` if the old behavior is required.
- Parsing unknown `TextDirection`, `TextAlign`, `Hyphens`, or `FontSynthesis`
  values throws `IllegalArgumentException`.
- Material components may now avoid display cutouts by default. Override the
  component inset when edge-to-edge content should occupy that region.

## High-value runtime and state APIs

### Retain values without serialization

Use `retain` for values that should survive temporary removal from the
composition but do not belong in serialized saveable state. Its lifetime is
shorter than saveable state. On Android, a lifecycle-aware retain scope also
survives configuration changes.

Keys are retained with their values, so do not use keys that hold resources
longer than intended. Mark types that must never be retained with
`@DoNotRetain`. Use `RetainedEffect` for retention-lifecycle work and install
custom `ManagedRetainedValuesStore` instances through
`LocalRetainedValuesStoreProvider`.

### Pause and diagnose composition

`PausableComposition` can pause a subcomposition and apply it asynchronously,
provided the compiler supports the feature. Inspect `isApplied` and
`isCancelled`; always dispose a cancelled instance instead of reusing it.

For production diagnostics, group-key stack traces work in minified builds
when mapping generation is enabled. The compiler plugin begins generating the
required mappings with Kotlin 2.3.0. Runtime tooling can also inspect the
experimental `RecomposerInfo.errorState`.

## High-value UI capabilities

### Scrolling, visibility, and input

- `Modifier.scrollable2D` and `Scrollable2DState` support two-axis scrolling;
  `canScroll` receives an `Offset`.
- `ScrollIndicatorState` is exposed by standard scroll states, while
  `Modifier.scrollIndicator` and `ScrollIndicatorFactory` support custom
  indicators.
- Prefer `onVisibilityChanged()` to deprecated `onFirstVisible()`, and track
  prior visibility yourself when an event must occur only once.
- Trackpad pan and scale events have dedicated pointer types and test
  injection support; cursor-driving trackpad gestures normally appear as
  mouse input.
- Typed semantic autofill uses `FillableData`, `fillableData`, and
  `onFillData`; semantic autofill behavior is always enabled.

### Layout and animation

- `Modifier.animateBounds` animates size and position in a lookahead scope;
  lazy grids and pagers participate in lookahead and approach passes.
- Stable shared-transition APIs support dynamic enablement, fallback target
  bounds, gesture velocity, lookahead coordinates, and skipping to a
  lookahead position.
- `FlexBox` covers row, column, and flow-style layouts with grow, shrink,
  wrapping, direction, and alignment controls.
- Experimental non-lazy `Grid` offers explicit two-dimensional tracks and
  item placement; use `MinMax(0.dp, 1.fr)` to avoid intrinsic measurement of
  flexible tracks containing subcomposition.

### Text and editing

- `TextAutoSize` supports custom autosizing. Start and middle ellipsis require
  a single line.
- State-backed fields support styled output through
  `OutputTransformation` and `TextFieldBuffer.addStyle`.
- Public text-context-menu APIs can append or filter components, including
  Android `PROCESS_TEXT` actions.
- `InputTextSuggestionState` and `TextCompositionRange` expose
  transliteration replacement and active composition state.

### Material 3

- Prefer state-backed search APIs: collapsed and expanded surfaces are
  separate, and stable slot-based APIs use `SearchBarState`.
- Hoist slider state with `rememberSliderState` or
  `rememberRangeSliderState`.
- Use `rememberTooltipPositionProvider`; modern tooltip dismissal is
  callback-based and `TooltipBox` is non-focusable by default.
- Keep Material 3's built-in themed ripple configuration. Add
  `material3-ripple` directly only when replacing a direct legacy ripple
  dependency or using inset focus rings without the full library.

## Applying this guidance

1. Identify the exact Compose artifact and resolved version behind the failing
   call, not just the BOM coordinate.
2. Fix toolchain and removed-API failures before debugging behavioral changes.
3. Search the topic reference for both the old and new API names; successive
   releases may first introduce a bridge and later remove it.
4. For behavior-gated migrations, check whether the flag still exists at the
   resolved version. Do not copy an obsolete opt-out into newer code.
5. Update tests after semantics, dispatcher, focus, inset, or host-theme
   changes; these can alter test observations without changing app intent.
