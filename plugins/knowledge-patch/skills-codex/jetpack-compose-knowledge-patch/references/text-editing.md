# Text and Editing

## Autosizing and overflow

`AutoSize` was renamed to `TextAutoSize` in 1.8.0, with public APIs for custom
autosizing implementations. Deprecated `AutoSize` overloads were removed;
migrate each call to its corresponding `TextAutoSize` API.

Single-line text supports `TextOverflow.StartEllipsis` and
`TextOverflow.MiddleEllipsis` in addition to end ellipsis. Keep
`maxLines = 1` when using either mode.

## Annotated strings and paragraphs

`Paragraph` and `ParagraphIntrinsics` receive every `AnnotatedString`
annotation, not only span styles (since 1.8.0). `AnnotatedString` permits fully
overlapping and nested paragraphs, its builder methods are stable, and
`AnnotatedString.fromHtml` supports `<ul>` and `<li>`.

Compose 1.9.0 adds `AnnotatedString` APIs for constructing custom bullet
lists.

## BasicText, cursors, and fonts

`BasicText` no longer adds an implicit `graphicsLayer` in 1.8.0. Add
`Modifier.graphicsLayer()` when code intentionally relies on the layer rather
than assuming one exists.

Tests can disable cursor rendering with `LocalCursorBlinkEnabled`.

A resource font that cannot load now silently falls back to the default font
instead of throwing during measurement. If a test expects a measurement-time
exception, assert the rendered fallback or validate the resource separately.

## State-backed field edits and undo

`TextFieldState.edit {}` creates a standalone undo entry in 1.9.0 instead of
clearing undo history. When a programmatic replacement should establish a new
history baseline, call:

```kotlin
textFieldState.undoState.clearHistory()
```

For rendered styling, an `OutputTransformation` can call
`TextFieldBuffer.addStyle`. The interim `AnnotatedOutputTransformation` API is
removed.

## Context menus and Android smart selection

Text fields support right-click context menus and Android smart-selection
items in 1.9.0. `ComposeFoundationFlags.isSmartSelectionEnabled` controls smart
selection, and `LocalTextClassifierCoroutineContext` controls its work
context.

Public customization uses:

- `Modifier.appendTextContextMenuComponents`
- `Modifier.filterTextContextMenuComponents`
- the text-context-menu provider, data, and component APIs
- `ProcessTextKey` for Android `PROCESS_TEXT` actions

## Secure text

`BasicSecureTextField` hoists the `ScrollState` used by its internal field in
1.9.0. `TextObfuscationMode.RevealLastTyped` follows Android's
`TEXT_SHOW_PASSWORD` system setting.

Material 3 supplies higher-level `SecureTextField` and
`OutlinedSecureTextField`; see the Material component reference for their
state-backed field family.

## Selection and transformed offsets

`SemanticsNodeInteraction.performTextInputSelection` is stable in 1.9.0. Its
`relativeToOriginal` argument determines whether selection offsets refer to
the original text or transformed output.

In 1.10.0, double-tap word selection works inside `SelectionContainer` and in
the value/on-value-change `BasicTextField` overload. Mouse-wheel support is
covered with scrolling because it also accepts two-dimensional deltas.

## Transliteration input

`InputTextSuggestionState` in 1.11.0 exposes the current replacement
suggestions supplied by a transliteration IME. `TextCompositionRange`
identifies the active transliteration composition; `null` means that no
composition is active.

## Common clipboard and tooltip APIs

Foundation and UI expose a common `Clipboard` interface and composition local
in 1.8.0. `BasicTooltip` is also available from common Foundation code, so
shared UI need not depend on the Material tooltip implementation.

## Parsing serialized text enums

In 1.10.0, `valueOf` for `TextDirection`, `TextAlign`, `Hyphens`, and
`FontSynthesis` throws `IllegalArgumentException` for unknown values. Validate
external or persisted strings before parsing, or handle that exception.
