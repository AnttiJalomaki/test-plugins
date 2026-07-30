# Text, Autofill, and Resources

## Migrate to semantics-based autofill

Compose text autofill requires both UI and Foundation 1.8.0 or newer. The old
autofill APIs are deprecated, and `AutofillManager` is now an abstract class.
`InputText` exposes the text before output transformation,
`requestAutofill` is no longer a manager method, the text toolbar can trigger
autofill, and `LocalAutofillHighlightColor` uses `Color` (1.8.0).

Autofill is typed from 1.10.0. `FillableData` supports text, Boolean, integer,
list, and date values. Its factories moved to the companion object, so call
`FillableData.createFrom(value)`. Read date data through `dateMillisValue`.
Use the `fillableData` property and `onFillData` action instead of deprecated
`onAutofillText`. A composition local can customize the highlight brush shown
after a successful fill.

Remove the old `isSemanticAutofillEnabled` Compose UI flag. Semantic autofill
is always enabled after its removal in 1.11.0.

## Autosize and truncate text

`AutoSize` was renamed to `TextAutoSize`, with public APIs for custom autosize
implementations. Deprecated `AutoSize` overloads were removed; move callers to
the corresponding `TextAutoSize` APIs (1.8.0).

Single-line text supports `TextOverflow.StartEllipsis` and
`TextOverflow.MiddleEllipsis` in addition to end ellipsis. Keep
`maxLines = 1` for either new mode (1.8.0).

## Preserve all annotated-string information

`Paragraph` and `ParagraphIntrinsics` receive every `AnnotatedString`
annotation rather than just span styles. `AnnotatedString` permits fully
overlapping and nested paragraphs, its builder methods are stable, and
`AnnotatedString.fromHtml` supports `<ul>` and `<li>` (1.8.0).

`AnnotatedString` also supplies custom bullet-list construction APIs
(1.9.0).

## Style state-backed text-field output

For a state-backed text field, `OutputTransformation` can style rendered
output through `TextFieldBuffer.addStyle`. The interim
`AnnotatedOutputTransformation` API was removed (1.9.0).

`TextFieldState.edit {}` creates an independent undo entry rather than
clearing undo history. Call `TextFieldState.undoState.clearHistory()` when a
programmatic edit should deliberately reset that history (1.9.0).

## Customize context menus and smart selection

Text fields support right-click context menus and Android smart-selection
items. Control smart selection with
`ComposeFoundationFlags.isSmartSelectionEnabled` and provide its work context
through `LocalTextClassifierCoroutineContext` (1.9.0).

Customize menu content through `Modifier.appendTextContextMenuComponents`,
`filterTextContextMenuComponents`, and the text-context-menu provider, data,
and component APIs. Use `ProcessTextKey` for Android `PROCESS_TEXT` actions
(1.9.0).

## Secure text and scrolling

`BasicSecureTextField` hoists the `ScrollState` used by its internal text
field. `TextObfuscationMode.RevealLastTyped` follows Android's
`TEXT_SHOW_PASSWORD` system setting (1.9.0).

## Handle transliteration suggestions

`InputTextSuggestionState` reports the current replacement suggestions from a
transliteration IME. `TextCompositionRange` identifies the active
transliteration composition range; `null` means no composition is active
(1.11.0).

## Account for BasicText rendering

`BasicText` no longer inserts an implicit `graphicsLayer`. Add
`Modifier.graphicsLayer()` when code depends on layer isolation or other layer
behavior. Tests can suppress cursor drawing through `LocalCursorBlinkEnabled`
(1.8.0).

## Handle missing fonts gracefully

A resource font that cannot load falls back silently to the default font
instead of throwing during measurement (1.8.0). Do not rely on a measurement
exception as the signal for a missing resource font.

## Access configuration-aware resources

Use `LocalResources.current` for Android resource reads that must react to
configuration changes. The read invalidates composition, so later lookups see
the new configuration (1.9.0).

## Use common clipboard and tooltip APIs

Foundation and UI expose a common `Clipboard` interface through a composition
local. `BasicTooltip` is also available to common Foundation code (1.8.0).

## Parse text enums defensively

`TextDirection`, `TextAlign`, `Hyphens`, and `FontSynthesis` `valueOf`
functions throw `IllegalArgumentException` for unknown input from 1.10.0.
Validate external strings or handle the exception.
