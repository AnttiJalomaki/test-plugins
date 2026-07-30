# Semantics, Accessibility, and Testing

## Migrate semantics APIs

Use `Role.Carousel` for pager-like controls. Replace
`invisibleToUser()` with `hideFromAccessibility()`. The
`SemanticsNodeInteraction.semanticsId()` helper was removed; obtain the ID
through `fetchSemanticsNode().id` (1.8.0).

`isHiddenFromAccessibility()` matches hidden semantics in tests (1.10.0).

## Avoid brittle semantics-tree assertions

`background`, `border`, and `graphicsLayer` modifier nodes implement
`SemanticsModifierNode`. They can insert nodes and invalidate tests that rely
on exact parent, child, or sibling relationships. Apply a test tag directly to
the intended node or use a looser ancestor matcher (1.9.0).

## Describe meaningful bounds and shapes

The `Shape` semantics property describes a control whose meaningful shape
differs from its bounding rectangle.
`SemanticsModifierNode.isImportantForBounds` can exclude a node from bounds
calculation. An Android-specific `SemanticsPropertyKey` factory publishes
custom values through `AccessibilityNodeInfo.getExtras` (1.9.0).

## Support nested accessibility scrolling

Accessibility `showOnScreen` actions can walk up through nested scrolling
containers. Off-screen semantics children of a partially visible merging node
remain exposed to accessibility services instead of disappearing from the
tree (1.11.0).

## Use the split accessibility-test artifacts

Use `enableAccessibilityChecks()` from
`compose:ui:ui-test-accessibility` when no test rule is involved. When calling
the API on a rule, use
`compose:ui:ui-test-junit4-accessibility`. The experimental
`GlobalAssertions` API was removed; use accessibility checks (1.8.0).

## Control the instrumented-test host theme

The `ComposeContentTestRule.setContent` host supplied by `ui-test-manifest`
uses `Theme.Material.Light.NoActionBar`, preventing an action bar from covering
test content when targeting SDK 35. To use another theme, remove
`ui-test-manifest` and declare `ComponentActivity` with the desired theme in
the test manifest (1.8.0).

## Select text by original or transformed offsets

`SemanticsNodeInteraction.performTextInputSelection` is stable. Its
`relativeToOriginal` parameter chooses whether offsets refer to the original
or transformed text (1.9.0).

## Run suspend UI tests and preserve failures

Experimental `runComposeUiTest` accepts a suspend block. Uncaught layout or
draw exceptions can be reported without terminating the complete suite
(1.9.0).

## Choose a scheduler intentionally

The `effectContext` variants of `createComposeRule`,
`createAndroidComposeRule`, and `createEmptyComposeRule` are stable and accept
a `StandardTestDispatcher`. Call `MainTestClock.runCurrent()` to execute due
scheduler work. The default for these original APIs remains
`UnconfinedTestDispatcher` (1.10.0).

Compose UI testing v2 consists of
`androidx.compose.ui.test.v2.run*ComposeUiTest` and
`androidx.compose.ui.test.junit4.v2.create*ComposeRule`. These APIs use
`StandardTestDispatcher` by default, so coroutines remain queued until
scheduled. Their shared `TestCoroutineScheduler` is exposed for calls such as
`runCurrent()` (1.11.0).

`ComposeUiTestFlags.isStandardTestDispatcherSupportEnabled` was removed.
Deprecated test variants continue to use `UnconfinedTestDispatcher`
(1.11.0).

## Respect test-result annotations

`SemanticsNode` finder and selector results carry `@CheckResult`; do not ignore
them accidentally (1.10.0).

## Inject mouse and trackpad behavior

Mouse-wheel tests can inject two-dimensional deltas through
`MouseInjectionScope` (1.10.0).

For trackpads, use `SemanticsNodeInteraction.performTrackpadInput` or
`MultiModalInjectionScope.trackpad`. Multimodal key and rotary injection APIs
are stable. Pan and scale end injection accepts `delayMillis` (1.11.0).

## Diagnose asynchronous failures

Experimental Compose diagnostic stack traces cover work launched by
`LaunchedEffect` and `rememberCoroutineScope` (1.9.0). Combine those traces
with the scheduler appropriate to the test API instead of assuming launched
coroutines run immediately.
