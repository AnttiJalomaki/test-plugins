---
name: ruff-knowledge-patch
description: Ruff
version: 0.16.0
license: MIT
metadata:
  author: Nevaberry
---


# Ruff Compatibility Guide

Use this skill when upgrading Ruff, changing lint or format configuration,
reviewing generated fixes, integrating Ruff with editors or CI, or consuming
Ruff's machine-readable output.

## Reference index

| Reference | Topics |
| --- | --- |
| [Configuration, CLI, and server](references/configuration-cli-and-server.md) | Python targets, rule selection, suppression syntax, CLI output, editor and server settings |
| [Rule lifecycle and selection](references/rule-lifecycle-and-selection.md) | Stable rules, preview rules, renamed, deprecated, and removed rules |
| [Rule behavior](references/rule-behavior.md) | Detection changes, expanded coverage, defaults, and rule-specific edge cases |
| [Formatting and syntax](references/formatting-and-syntax.md) | Stable and preview style changes, syntax validation, Markdown, f-strings |
| [Fixes and safety](references/fixes-and-safety.md) | Fix availability, safe/unsafe reclassification, comment preservation |
| [Analysis, discovery, and distribution](references/analysis-discovery-and-distribution.md) | Dependency graphs, import classification, file discovery, containers, binaries, source builds |

## Upgrade triage

Check these areas before accepting a Ruff upgrade:

1. Pin `target-version` if the project cannot tolerate changing implicit Python
   baselines.
2. Search configuration and suppression comments for renamed, split,
   deprecated, or removed rule codes.
3. Make the selected rule set explicit when relying on preview defaults or
   prefix selection.
4. Run formatting separately and review the resulting diff, especially for
   f-strings, `match`, `with`, lambdas, method chains, and Markdown.
5. Review unsafe fixes and any fix that can remove comments, change expression
   types, or add imports.
6. Update output parsers for nullable JSON locations and changed human-readable
   check output.
7. Check editor settings, dependency-graph inputs, container bases, and
   source-build requirements where applicable.

## Breaking rule-code and lifecycle changes

Update exact selections, ignores, and suppression comments when codes move:

| Old selection | Replacement or action |
| --- | --- |
| `RUF035` (`unsafe-markup-use`) | Use `S704` |
| `RUF025` | Use `RUF037` |
| `UP007` for `Optional` | Use `UP045`; `UP007` continues to cover `Union` |
| old Airflow `AIR301` | Use `AIR002` |
| old Airflow `AIR302` | Use `AIR301` |
| old Airflow `AIR303` | Use the reorganized `AIR302`, `AIR311`, or `AIR312` check as applicable |
| `S320` | Removed; do not select or suppress it |
| `PD901` | Removed; do not select or suppress it |
| `UP038` | Removed; do not select or suppress it |

Deprecated rules are not enabled by a group or prefix selection. While a
deprecated rule still exists, select it by its exact code. Remove the selection
once the rule is removed.

`A005` is named `stdlib-module-shadowing`; it was previously called
`builtin-module-shadowing`.

## Rule-selection defaults

Do not assume preview mode preserves the small stable default selection.
Preview broadened its default substantially and adjusted the set during the
0.15 series. Ruff 0.16 enables 413 rules by default, but the set is not a strict
superset of the former 59-rule default.

Pin the former core selection when that is the intended policy:

```toml
[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
```

Pin every policy-critical rule or prefix rather than relying on an evolving
implicit set.

## Python target configuration

Pin the project's real Python baseline:

```toml
[tool.ruff]
target-version = "py312"
```

The announced inference from `requires-python` was deferred from 0.10.0 and
first arrived in 0.11.0. Later releases also changed the Python version used
when no target applies. Syntax checks and ordinary lint decisions have not
always used the same fallback.

Ruff recognizes `py314`. Preview mode recognizes `py315`, including Python 3.15
lazy imports and starred unpacking of comprehensions.

When configuration uses `extend`, the complete inheritance chain is resolved
before the default Python version is chosen. An inherited target can therefore
override what previously appeared to be an earlier fallback.

## Suppression migration

Use the canonical spaced form for the newer suppression syntax:

```python
import math  # ruff: ignore[F401]

# ruff: ignore[F401]
import os
```

Block `ruff:disable` and `ruff:enable` controls are stable. Preview additionally
supports file-level, logical-line, and nested logical-line suppression forms,
plus human-readable rule names.

File-level and inline `noqa` comments share one parser. Fix malformed comments
that the unified parser rejects. `PGH004` checks blanket file-level `noqa`, and
`RUF100` finds unused file-level and range suppressions.

## Formatting migration

Treat formatter output changes as expected upgrade work:

- The stable 2025 style changes f-strings, implicit concatenation, assertions,
  `match`, return annotations, comprehensions, older-Python `with` statements,
  and dynamic-width docstring blocks.
- The stable 2026 style incorporates method-chain, lambda, exception-type,
  `match`, triple-quote, and stub-class spacing changes.
- Python 3.13.4-compatible formatting avoids a now-invalid break after a
  multiline f-string format specifier.
- Markdown Python blocks are formatted by default in 0.16; an upgrade can
  therefore modify documentation as well as Python files.

Run the formatter in a dedicated change and inspect the diff before mixing it
with functional edits.

To format files while making the command fail when it changed anything:

```console
ruff format --exit-non-zero-on-format .
```

For CI annotation output:

```console
ruff format --check --output-format github .
```

## Fix safety

Never equate “a fix exists” with “the fix is behavior-preserving.” Ruff marks
many fixes unsafe when they can:

- remove comments;
- alter an expression or return type;
- rewrite non-literal or otherwise weakly constrained input;
- add or remove imports;
- change class behavior; or
- affect code outside typing-only contexts.

Review the complete safety tables before bulk-applying fixes. Particularly
important cases include `FURB171` with a string right-hand side, `RUF046`
parenthesization, `UP007` and `UP045` annotation rewrites, pathlib conversions,
and rules whose fixes delete comments.

With `lint.future-annotations = true`, typing-related fixes can insert:

```python
from __future__ import annotations
```

That applies to fixes involving `TC001`, `TC002`, `TC003`, `RUF013`, and
`UP037`, and in preview also to `UP006`, `UP007`, and `UP045`.

Disable generated `typing_extensions` imports when they are not allowed:

```toml
[tool.ruff.lint]
typing-extensions = false
```

## Output and integration checks

Ruff's human-readable check output can now include fix diffs. Avoid parsing it
as a stable machine interface.

JSON consumers must accept `null` for:

- `filename`;
- `location` and `end_location`; and
- `fix.edits[].location` and `fix.edits[].end_location`.

Server logging is controlled only by `logLevel`, whose default is `info`; LSP
`trace` does not enable or disable logging. `ruff.printDebugInformation` no
longer emits log output. Code actions ignore diagnostics from other sources.

`ruff check --watch` respects `--output-format` and defaults to `full`.

## Focused configuration changes

Replace deprecated `flake8-builtins` option names:

```toml
[tool.ruff.lint.flake8-builtins]
allowed-modules = ["builtins"]
```

Use unprefixed option names such as `allowed-modules`; the older
`builtins-allowed-modules` form is deprecated. Also note that
`lint.flake8-builtins.strict-checking` now defaults to `false`.

On macOS, put user configuration in the XDG location, normally
`~/.config/ruff/ruff.toml`; the old `~/Library/Application Support/ruff/ruff.toml`
fallback is gone.

Map Quarto files explicitly when they should be treated as Markdown:

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

## Verification workflow

Use an upgrade branch and run:

```console
ruff check .
ruff format --check .
```

Then:

1. Compare the effective rule selection with the project's intended policy.
2. Search for old rule codes and malformed suppression comments.
3. Re-run with preview only if preview behavior is intentionally in scope.
4. Apply fixes in reviewed groups, separating unsafe changes.
5. Re-run project tests after lint and format changes.
6. Validate editor, CI annotation, JSON, and dependency-graph consumers.

Consult the topic references for the complete rule lists and edge cases; the
quick reference intentionally emphasizes migration hazards.
