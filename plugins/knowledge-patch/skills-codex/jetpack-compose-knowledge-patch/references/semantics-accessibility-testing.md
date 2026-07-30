# Semantics, Accessibility, and Testing

## Semantics API migrations

In 1.8.0:

- Use `Role.Carousel` for pager-like controls.
- Replace `invisibleToUser()` with `hideFromAccessibility()`.
- `SemanticsNodeInteraction.semanticsId()` is removed; use
  `fetchSemanticsNode().id`.

The 1.10.0 test matcher `isHiddenFromAccessibility()` recognizes the hidden
semantics state.

## Semantics-tree structure

The `background`, `border`, and `graphicsLayer` modifier nodes implement
`SemanticsModifierNode` in 1.9.0. They can insert semantics nodes and break
tests that depend on an exact parent, child, or sibling structure. Put the tag
on the intended node itself or use a looser ancestor-based matcher.

The `Shape` semantics property describes a control whose meaningful shape
differs from its bounding rectangle. `SemanticsModifierNode.isImportantForBounds`
can exclude a node from bounds calculation. On Android, an Android-specific
`SemanticsPropertyKey` factory publishes custom values through
`AccessibilityNodeInfo.getExtras`.

## Accessibility scrolling and merged nodes

In 1.11.0, accessibility `showOnScreen` actions can walk up nested scrolling
containers. Off-screen semantics children of a partially visible merging node
remain exposed to accessibility services instead of disappearing from the
tree.

## Accessibility-check dependencies

As of 1.8.0, `enableAccessibilityChecks()` lives in different artifacts
according to how it is invoked:

- Use `compose:ui:ui-test-accessibility` when calling it without a test rule.
- Use `compose:ui:ui-test-junit4-accessibility` when invoking it on a rule.

The experimental `GlobalAssertions` API is removed; use accessibility checks
instead.

## Instrumented-test host theme

The `ComposeContentTestRule.setContent` host supplied by `ui-test-manifest`
uses `Theme.Material.Light.NoActionBar` in 1.8.0. This prevents an action bar
from covering test content when targeting SDK 35.

To use another theme, remove `ui-test-manifest` and declare
`ComponentActivity` with the desired theme in the test manifest.

## Text and general UI-test APIs

`SemanticsNodeInteraction.performTextInputSelection` is stable in 1.9.0. Set
`relativeToOriginal` according to whether offsets refer to original or
transformed text.

The experimental `runComposeUiTest` accepts a suspend block. Uncaught layout
or draw exceptions can be reported without terminating the entire test suite.

In 1.10.0, `SemanticsNode` finder and selector results carry `@CheckResult`,
making ignored results visible to static analysis.

## Dispatcher scheduling

The stable `effectContext` variants of `createComposeRule`,
`createAndroidComposeRule`, and `createEmptyComposeRule` accept a
`StandardTestDispatcher` in 1.10.0. Call `MainTestClock.runCurrent()` to run
work already due on its scheduler. Those APIs otherwise retain
`UnconfinedTestDispatcher` as the default.

Compose UI testing v2 in 1.11.0 changes the default:

- `androidx.compose.ui.test.v2.run*ComposeUiTest` and
  `androidx.compose.ui.test.junit4.v2.create*ComposeRule` use
  `StandardTestDispatcher` by default.
- Coroutines stay queued until their scheduler advances.
- The shared `TestCoroutineScheduler` is exposed for operations such as
  `runCurrent()`.
- Deprecated test variants keep `UnconfinedTestDispatcher`.
- `ComposeUiTestFlags.isStandardTestDispatcherSupportEnabled` is removed.

When migrating to v2, advance the shared scheduler explicitly rather than
assuming launched work has run inline.

## State restoration tests

`StateRestorationTester` always applies platform-specific state encoding in
1.10.0. Tests should use state that the target platform can encode instead of
depending on an earlier test-only representation.

## Trackpad and multimodal injection

Compose 1.11.0 tests can inject trackpad events with
`SemanticsNodeInteraction.performTrackpadInput` or
`MultiModalInjectionScope.trackpad`. Multimodal key and rotary injection are
stable. Pan and scale end events accept `delayMillis`, which is useful when
testing gesture-timing behavior.
