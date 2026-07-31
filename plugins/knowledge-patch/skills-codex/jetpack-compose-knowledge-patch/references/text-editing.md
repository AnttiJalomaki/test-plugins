# Text and Editing

## Display text

### Text autosizing (1.8.0)

`AutoSize` was renamed to `TextAutoSize`, including public APIs for custom
autosizing implementations. Deprecated `AutoSize` overloads were removed; use
the matching `TextAutoSize` APIs.

### Start and middle ellipsis (1.8.0)

Single-line text supports `TextOverflow.StartEllipsis` and
`TextOverflow.MiddleEllipsis` as well as end ellipsis. Set `maxLines = 1` when
using either new mode.

### Annotated strings and paragraphs (1.8.0)

`Paragraph` and `ParagraphIntrinsics` receive every `AnnotatedString`
annotation, not only span styles. Paragraph annotations may fully overlap or
nest. Builder methods are stable, and `AnnotatedString.fromHtml` supports
`<ul>` and `<li>`.

### `BasicText` rendering behavior (1.8.0)

`BasicText` no longer inserts an implicit `graphicsLayer`. Add
`Modifier.graphicsLayer()` when code depends on layer behavior. Tests can turn
off cursor drawing with `LocalCursorBlinkEnabled`.

### Resource-font failures (1.8.0)

A resource font that cannot load falls back silently to the default font
instead of throwing during measurement. If a particular font is required,
validate availability rather than treating successful measurement as proof.

## State-backed editing and undo

### Undo history (1.9.0)

`TextFieldState.edit {}` creates an independent undo entry; it no longer clears
history. When a programmatic replacement intentionally starts a new history,
call `TextFieldState.undoState.clearHistory()` explicitly.

### Styled output (1.9.0)

An `OutputTransformation` can style state-backed text output with
`TextFieldBuffer.addStyle`. The interim `AnnotatedOutputTransformation` API was
removed. `AnnotatedString` also supports custom bullet-list construction.

## Context menus and selection

### Context menus and smart selection (1.9.0)

Text fields support right-click context menus and Android smart-selection
items. On versions that expose them, control smart selection with
`ComposeFoundationFlags.isSmartSelectionEnabled` and its work context with
`LocalTextClassifierCoroutineContext`.

Customize public menus with:

- `Modifier.appendTextContextMenuComponents`
- `Modifier.filterTextContextMenuComponents`
- text-context-menu provider, data, and component APIs
- `ProcessTextKey` for Android `PROCESS_TEXT` actions

### Transformed selection offsets (1.9.0)

`SemanticsNodeInteraction.performTextInputSelection` is stable. Its
`relativeToOriginal` parameter chooses whether selection offsets refer to the
original or transformed text.

### Word selection (1.10.0)

Double-tap word selection works in `SelectionContainer` and in the
value/`onValueChange` overload of `BasicTextField`.

## Secure and IME-driven text

### Secure text behavior (1.9.0)

`BasicSecureTextField` hoists the `ScrollState` used internally.
`TextObfuscationMode.RevealLastTyped` follows Android's `TEXT_SHOW_PASSWORD`
system setting.

### Transliteration suggestion state (1.11.0)

`InputTextSuggestionState` exposes replacement-suggestion state from
transliteration IMEs. `TextCompositionRange` identifies the active
transliteration composition range; `null` means no active composition.

## Text parsing

### Strict `valueOf` behavior (1.10.0)

`TextDirection.valueOf`, `TextAlign.valueOf`, `Hyphens.valueOf`, and
`FontSynthesis.valueOf` throw `IllegalArgumentException` for unknown values.

## Editing checklist

- Remember that a programmatic `TextFieldState.edit` creates a standalone undo
  entry; call `TextFieldState.undoState.clearHistory()` when it should reset the
  undo stack.
- Use `OutputTransformation` and `TextFieldBuffer.addStyle` to style rendered
  output in state-backed text fields.
- Interpret selection offsets in the correct original or transformed space.
- Respect the host's password reveal setting for secure fields.
- Handle missing fonts and invalid serialized enum-like values explicitly when
  correctness depends on them.
