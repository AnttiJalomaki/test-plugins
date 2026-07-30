# Formatting and syntax

Use this reference to explain upgrade diffs, preview formatting, Python syntax
support, and version-sensitive parser behavior.

## Stable formatter styles

### Core expression and layout changes

The 2025 stable style in 0.9.0 can produce new diffs. It:

- formats expressions inside f-string elements, selects alternate quotes for
  strings inside f-strings, and preserves hexadecimal casing in debug
  expressions;
- chooses quote style per literal in an implicitly concatenated f-string and
  joins an implicitly concatenated string into one literal when it fits on one
  line;
- removes the `ISC001` incompatibility warning;
- prefers parentheses around an `assert` message instead of breaking the
  assertion expression;
- automatically parenthesizes overlong `if` guards in `match` cases and
  formats `match` patterns more consistently;
- removes unnecessary parentheses around return type annotations;
- keeps an opening parenthesis on the same line as `if` in a comprehension
  whose condition starts with a comment;
- formats single-context-manager `with` statements more consistently on
  Python 3.8 and older; and
- accounts correctly for line width in docstring code blocks when
  `max-doc-code-line-length = "dynamic"`.

The 0.10.0-guide formatter no longer adds unnecessary parentheses around a
single-context-manager `with` statement with a trailing comment.

In 0.11.0 preview formatting stopped adding trailing whitespace to a docstring
after an escaped quote. Match cases using `[]` and `_` also became consistent.

The 0.14.0 preview style:

- removes parentheses around multiple exception types on Python 3.14+;
- permits newlines after function headers without docstrings;
- avoids extra parentheses around long `match` patterns with `as` captures;
- adds fluent method-chain formatting; and
- keeps lambda parameters on one line while parenthesizing an expanded body,
  with later corrections preserving required lambda-body parentheses.

Those preview changes became the stable 2026 style in 0.15.0. Stable formatting
also permits omission of extra spaces between escaped quotes and closing triple
quotes, enforces blank lines before decorated classes in stubs, and adds the
`nested-string-quote-style` option in 0.15.9.

## F-string compatibility

Python 3.13.4 made a line break after a format specifier in a multiline
f-string a syntax error. Ruff 0.12.0 avoids inserting that break, so an upgrade
can reformat affected f-strings.

`py314` is accepted as a target in 0.11.0, including preview handling for
deferred annotations and Python 3.14 unparenthesized exception tuples. By
0.11.13, the parser and formatter also support template strings:

```toml
[tool.ruff]
target-version = "py314"
```

## Compile-time syntax validation

Preview checking in 0.11.0 reports compile-time errors including:

- duplicate parameters or type parameters;
- invalid `match` patterns;
- illegal starred expressions;
- invalid annotations;
- module-level `nonlocal`; and
- assignment to or deletion of `__debug__`.

It also version-gates PEP 701 f-strings, parenthesized context managers, starred
annotations and indexes, tuple unpacking in `for` iterators, and
unparenthesized exception tuples. When no Python version applies to a
version-related preview diagnostic, this behavior initially used the latest
supported Python version.

In 0.12.0 these checks moved into regular checking. They include CPython
compiler errors such as an irrefutable `match` pattern before the final case.
With no `target-version`, version-related syntax checks assumed the latest
supported Python, 3.13, while normal lint-rule behavior still assumed the
minimum supported Python, 3.9.

## Python 3.15 preview syntax

Preview parsing in 0.15.0 supports Python 3.15 lazy imports and PEP 798
star-unpacking of comprehensions. It validates lazy-import syntax and preserves
the `lazy` keyword during import sorting.

`TID254` can require or prohibit lazy imports, another preview check detects a
lazy import evaluated eagerly, and `RUF017` uses starred unpacking on Python
3.15 and newer.

## Markdown formatting

Preview formatting in 0.15.0 processes Python code blocks labeled with Python,
`pycon`, or Quarto language markers; unlabeled blocks remain untouched. The
language server supports the same formatting. Markdown files became discoverable
by default in preview in 0.15.5.

Quarto `.qmd` files lost their implicit special case. Map them explicitly if
they should be treated as Markdown:

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

Configured extension mappings affect file discovery, code-block language
selection, and server handling. In 0.16.0-guide Markdown formatting became a
default behavior, so `ruff format` can modify Markdown without preview mode.

## Diagnosing unexpected formatting

1. Confirm the Ruff version and whether preview is active.
2. Pin `target-version` rather than relying on changing implicit defaults.
3. Check the full `extend` chain for an inherited target version.
4. Include Markdown and configured extensions in the input inventory.
5. Distinguish an intentional stable-style transition from a parser fix, such
   as the Python 3.13.4 f-string correction.
6. Run `ruff format --check` before writing and inspect Python and Markdown
   diffs independently.
