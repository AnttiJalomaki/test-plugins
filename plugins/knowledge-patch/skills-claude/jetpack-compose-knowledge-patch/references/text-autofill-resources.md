# Text, Autofill, and Resources

## Semantic autofill

### Foundation and UI migration (1.8.0)

Text autofill requires both Compose UI and Foundation 1.8 or newer. Migrate
from the deprecated legacy autofill APIs to semantics-based autofill.

`AutofillManager` is now an abstract class. `InputText` exposes the value
before output transformation. `requestAutofill` is no longer a manager
method, and the text toolbar can trigger autofill.
`LocalAutofillHighlightColor` now uses `Color`.

### Typed fill data (1.10.0)

`FillableData` supports text, Boolean, integer, list, and date values. Its
factories moved to the companion object, so call
`FillableData.createFrom(value)`. Read date data through `dateMillisValue`.

Use the `fillableData` semantics property and `onFillData` action instead of
deprecated `onAutofillText`. A composition local can customize the highlight
brush shown after a successful fill.

## Text sizing and overflow

### Autosizing (1.8.0)

`AutoSize` was renamed to `TextAutoSize`, with public APIs for custom sizing
implementations. Deprecated `AutoSize` overloads were removed; migrate every
call to its corresponding `TextAutoSize` API.

### Start and middle ellipsis (1.8.0)

Single-line text supports `TextOverflow.StartEllipsis` and
`TextOverflow.MiddleEllipsis` as well as end ellipsis. Keep `maxLines = 1`
when selecting either new mode.

## Annotated text and output styling

### Paragraph annotations and HTML lists (1.8.0)

`Paragraph` and `ParagraphIntrinsics` receive every `AnnotatedString`
annotation, not only span styles. `AnnotatedString` permits fully overlapping
and nested paragraphs, its builder methods are stable, and
`AnnotatedString.fromHtml` supports `<ul>` and `<li>`.

### Styled output and bullets (1.9.0)

For a state-backed text field, `OutputTransformation` can style rendered
output with `TextFieldBuffer.addStyle`. The interim
`AnnotatedOutputTransformation` API was removed.

`AnnotatedString` also has APIs for constructing custom bullet lists.

## Editing and undo

### Programmatic edits (1.9.0)

`TextFieldState.edit {}` creates a standalone undo entry instead of clearing
history. When a programmatic edit should reset undo, call
`TextFieldState.undoState.clearHistory()` explicitly.

## Context menus and smart selection

### Public customization (1.9.0)

Text fields support right-click context menus and Android smart-selection
items. Control smart selection with
`ComposeFoundationFlags.isSmartSelectionEnabled` and its work context with
`LocalTextClassifierCoroutineContext`.

Customize menus with `Modifier.appendTextContextMenuComponents`,
`filterTextContextMenuComponents`, and the text-context-menu provider, data,
and component APIs. Use `ProcessTextKey` for Android `PROCESS_TEXT` actions.

## Secure text

### Scroll state and system reveal preference (1.9.0)

`BasicSecureTextField` hoists the `ScrollState` used by its internal text
field. `TextObfuscationMode.RevealLastTyped` follows Android's
`TEXT_SHOW_PASSWORD` system setting.

## Rendering and selection behavior

### BasicText and cursor rendering (1.8.0)

`BasicText` no longer adds an implicit `graphicsLayer`; add
`Modifier.graphicsLayer()` when a layer is required. Tests can turn off cursor
drawing through `LocalCursorBlinkEnabled`.

### Double-tap selection (1.10.0)

Double-tap word selection works in `SelectionContainer` and in the
value/`onValueChange` form of `BasicTextField`.

## IME transliteration

### Suggestion and composition state (1.11.0)

`InputTextSuggestionState` exposes replacement-suggestion state from
transliteration IMEs. `TextCompositionRange` identifies the active
transliteration composition range; `null` means no composition is active.
