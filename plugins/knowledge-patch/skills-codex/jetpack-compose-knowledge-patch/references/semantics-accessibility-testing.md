# Semantics, Accessibility, and Testing

## Semantic autofill

### Semantics-based autofill (1.8.0)

Compose autofill is semantics-based. Text autofill requires both UI and
Foundation 1.8 or newer. The old autofill APIs are deprecated.
`AutofillManager` is an abstract class, `InputText` exposes the pre-output-
transformation value, and `requestAutofill` is no longer a manager method. The
text toolbar can initiate autofill. `LocalAutofillHighlightColor` uses `Color`.

### Typed fill data (1.10.0)

`FillableData` supports text, Boolean, integer, list, and date values. Its
factories live on the companion object, so call `FillableData.createFrom(value)`;
read dates through `dateMillisValue`. Replace deprecated `onAutofillText` with
the `fillableData` property and `onFillData` action. A composition local can
customize the highlight brush for a successful fill.

## Semantics API migrations

### Roles, hidden content, and node IDs (1.8.0)

Use `Role.Carousel` for pager-like controls. Replace `invisibleToUser()` with
`hideFromAccessibility()`. `SemanticsNodeInteraction.semanticsId()` was
removed; obtain the ID from `fetchSemanticsNode().id`.

### Decoration modifiers can insert semantics nodes (1.9.0)

The `background`, `border`, and `graphicsLayer` nodes implement
`SemanticsModifierNode`. They can insert nodes and change exact parent, child,
or sibling relationships. Tag the intended target directly or use a resilient
ancestor-based matcher instead of hard-coding the full tree shape.

### Bounds, shapes, and Android extras (1.9.0)

The `Shape` semantics property describes meaningful control shapes that differ
from their bounding rectangle. Set `SemanticsModifierNode.isImportantForBounds`
to exclude a node from bounds calculation. On Android, a platform-specific
`SemanticsPropertyKey` factory exposes custom values through
`AccessibilityNodeInfo.getExtras`.

## Accessibility behavior

### Accessibility checks artifacts (1.8.0)

For checks without a rule, `enableAccessibilityChecks()` is in
`compose:ui:ui-test-accessibility`. For checks invoked on a rule, use
`compose:ui:ui-test-junit4-accessibility`. The experimental `GlobalAssertions`
API was removed; migrate to accessibility checks.

### Nested show-on-screen and off-screen children (1.11.0)

Accessibility `showOnScreen` actions traverse nested scrolling containers.
Off-screen semantics children of a partially visible merging node remain
reported to accessibility services rather than disappearing from the exposed
tree.

## Test hosts and themes

### `ui-test-manifest` host theme (1.8.0)

The `ComposeContentTestRule.setContent` host supplied by `ui-test-manifest`
uses `Theme.Material.Light.NoActionBar`, preventing an action bar from covering
test content at target SDK 35. To use another theme, remove
`ui-test-manifest` and declare `ComponentActivity` with the desired theme in
the test manifest.

## Test APIs and scheduling

### Selection, suspend tests, and nonfatal errors (1.9.0)

`SemanticsNodeInteraction.performTextInputSelection` is stable;
`relativeToOriginal` chooses whether offsets use original or transformed text.
Experimental `runComposeUiTest` accepts a suspending block. The harness can
report uncaught layout or draw exceptions without ending the entire test suite.

### Scheduling and state restoration (1.10.0)

The `effectContext` overloads of `createComposeRule`,
`createAndroidComposeRule`, and `createEmptyComposeRule` are stable and accept a
`StandardTestDispatcher`. Call `MainTestClock.runCurrent()` to run due
scheduler work. The default for these APIs remains `UnconfinedTestDispatcher`.

`StateRestorationTester` always applies platform-specific state encoding.
`isHiddenFromAccessibility()` matches hidden semantics. Results from
`SemanticsNode` finders and selectors carry `@CheckResult`; consume them.

### Compose UI testing v2 (1.11.0)

The `androidx.compose.ui.test.v2.run*ComposeUiTest` and
`androidx.compose.ui.test.junit4.v2.create*ComposeRule` APIs use
`StandardTestDispatcher` by default. Coroutines remain queued until scheduled.
Use the exposed shared `TestCoroutineScheduler`, including `runCurrent()` when
due tasks must run.

`ComposeUiTestFlags.isStandardTestDispatcherSupportEnabled` was removed.
Deprecated test variants continue to use `UnconfinedTestDispatcher`.

## Test checklist

- Add the accessibility artifact that matches rule or no-rule usage.
- Match semantics by stable properties rather than exact decoration-node shape.
- Advance the correct scheduler after triggering effects or queued work.
- Test selection using the intended original or transformed offset space.
- Re-run restoration tests where platform encoding changes serialized values.
- Exercise `showOnScreen` through every nested scrolling parent.
