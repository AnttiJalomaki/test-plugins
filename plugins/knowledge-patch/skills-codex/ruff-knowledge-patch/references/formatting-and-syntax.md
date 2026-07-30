# Formatting and Syntax

## Stable 2025 formatter style

Ruff's 2025 stable style can create formatter diffs on upgrade. It includes
(0.9.0):

- formatting expressions inside f-string elements;
- choosing alternate quotes for strings inside f-strings;
- preserving hexadecimal casing in f-string debug expressions;
- choosing quote style per literal in an implicitly concatenated f-string;
- joining an implicitly concatenated string into one literal when it fits on
  one line;
- removing the `ISC001` incompatibility warning;
- preferring parentheses around an `assert` message instead of breaking the
  assertion expression;
- automatically parenthesizing overlong `if` guards in `match` cases and
  formatting `match` patterns more consistently;
- removing unnecessary parentheses around return type annotations;
- keeping the opening parenthesis on the same line as `if` in a comprehension
  whose condition has a leading comment;
- formatting single-context-manager `with` statements consistently on Python
  3.8 and older; and
- accounting correctly for line width in docstring code blocks when
  `max-doc-code-line-length = "dynamic"`.

The formatter later stopped adding unnecessary parentheses around a
single-context-manager `with` statement with a trailing comment
(0.10.0-guide).

## F-string, docstring, and match-case corrections

Preview formatting no longer leaves trailing whitespace in a docstring after
an escaped quote. Match cases using `[]` and `_` format consistently
(0.11.0).

Python 3.13.4 made a line break after the format specifier in a multiline
f-string a syntax error. Ruff avoids inserting that break, which can change
formatting on upgrade (0.12.0).

## Preview style leading to the stable 2026 formatter

Preview formatting removes parentheses around multiple exception types on
Python 3.14 and newer, permits newlines after function headers without
docstrings, and avoids extra parentheses around long `match` patterns with
`as` captures. It adds fluent method-chain formatting and keeps lambda
parameters on one line while parenthesizing an expanded body. Follow-up fixes
preserve parentheses required by a lambda body (0.14.0).

These 0.14-series preview changes became the stable 2026 style. Stable behavior
also allows omission of extra spaces between escaped quotes and closing triple
quotes, and enforces blank lines before decorated classes in stub files
(0.15.0).

## Syntax validation

Preview mode reports compile-time errors including (0.11.0):

- duplicate parameters or type parameters;
- invalid `match` patterns;
- illegal starred expressions;
- invalid annotations;
- module-level `nonlocal`; and
- assignment to or deletion of `__debug__`.

It version-gates PEP 701 f-strings, parenthesized context managers, starred
annotations and indexes, tuple unpacking in `for` iterators, and
unparenthesized exception tuples. When no Python version applies to a
version-related diagnostic, preview defaults to the latest supported version
(0.11.0).

These syntax checks became regular checks, including CPython compiler errors
such as an irrefutable `match` pattern before the final `case`. At that point,
an unset target made version-related syntax checks assume Python 3.13, while
other lint behavior still defaulted to Python 3.9 (0.12.0).

## Python 3.15 syntax and imports

Preview parsing supports Python 3.15 lazy imports and PEP 798 starred unpacking
of comprehensions. It validates lazy-import syntax and preserves the `lazy`
keyword during import sorting. `TID254` can require or ban lazy imports,
another preview check finds lazy imports evaluated eagerly, and `RUF017` uses
starred unpacking on Python 3.15 and newer (0.15.0).

## Markdown and mapped extensions

Preview formatting processes labeled Python code blocks in Markdown, including
`pycon` and Quarto language markers, while leaving unlabeled blocks alone. The
language server uses the same formatting. Markdown is discovered by default in
preview from 0.15.5. `.qmd` no longer has an implicit special case and must be
mapped explicitly (0.15.0):

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

Configured extensions participate in discovery, code-block language
selection, and later server handling (0.15.0).

Python code blocks in Markdown are now formatted by default, without a preview
opt-in, so `ruff format` can modify Markdown content (0.16.0-guide).
