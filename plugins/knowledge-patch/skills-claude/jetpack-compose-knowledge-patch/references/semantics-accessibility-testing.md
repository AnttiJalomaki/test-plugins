# Semantics, Accessibility, and Testing

## Semantics migrations

### Roles, visibility, and IDs (1.8.0)

Use `Role.Carousel` for pager-like controls. Replace
`invisibleToUser()` with `hideFromAccessibility()`.

`SemanticsNodeInteraction.semanticsId()` was removed. Retrieve the ID from
`fetchSemanticsNode().id`.

### Tree structure changes (1.9.0)

The `background`, `border`, and `graphicsLayer` modifier nodes implement
`SemanticsModifierNode`. They can insert nodes and break tests that depend on
exact parent, child, or sibling structure. Tag the intended node directly or
use a looser ancestor-based matcher.

### Shapes, bounds, and Android extras (1.9.0)

The `Shape` semantics property describes controls whose meaningful shape is
not their bounding rectangle. Set `SemanticsModifierNode.isImportantForBounds`
to `false` to exclude a node from bounds computation.

An Android-specific `SemanticsPropertyKey` factory exposes custom property
values through `AccessibilityNodeInfo.getExtras`.

## Accessibility behavior

### Nested show-on-screen (1.11.0)

Accessibility `showOnScreen` actions can walk upward through nested scrolling
containers. Off-screen semantics children of partially visible merging nodes
remain exposed to accessibility services instead of disappearing from the
tree.

## Accessibility checks

### Test artifacts (1.8.0)

When calling `enableAccessibilityChecks()` without a test rule, depend on
`compose:ui:ui-test-accessibility`. When invoking it on a JUnit 4 rule, depend
on `compose:ui:ui-test-junit4-accessibility`.

The experimental `GlobalAssertions` API was removed; use accessibility checks
instead.

## Test hosting

### Host theme (1.8.0)

The `ComposeContentTestRule.setContent` host supplied by `ui-test-manifest`
uses `Theme.Material.Light.NoActionBar`. This keeps an action bar from covering
test content when targeting SDK 35.

To choose another theme, remove `ui-test-manifest` and declare
`ComponentActivity` with the desired theme in the test manifest.

## Test APIs and scheduling

### Selection and runner updates (1.9.0)

`SemanticsNodeInteraction.performTextInputSelection` is stable. Its
`relativeToOriginal` parameter controls whether offsets refer to original or
transformed text.

Experimental `runComposeUiTest` accepts a suspend block. Uncaught layout or
draw exceptions can be reported without terminating the entire test suite.

### Stable dispatcher configuration (1.10.0)

The `effectContext` variants of `createComposeRule`,
`createAndroidComposeRule`, and `createEmptyComposeRule` are stable and accept
a `StandardTestDispatcher`. Call `MainTestClock.runCurrent()` to execute due
scheduler work. The default for these established APIs remains
`UnconfinedTestDispatcher`.

`StateRestorationTester` always applies platform-specific state encoding.
Use `isHiddenFromAccessibility()` to match hidden semantics. `SemanticsNode`
finder and selector results carry `@CheckResult`, so consume their return
values.

### Compose UI testing v2 (1.11.0)

The `androidx.compose.ui.test.v2.run*ComposeUiTest` and
`androidx.compose.ui.test.junit4.v2.create*ComposeRule` APIs use
`StandardTestDispatcher` by default. Coroutines remain queued until scheduled.

The shared `TestCoroutineScheduler` is exposed for calls such as
`runCurrent()`. `ComposeUiTestFlags.isStandardTestDispatcherSupportEnabled`
is removed. Deprecated test variants retain `UnconfinedTestDispatcher`.
