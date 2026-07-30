# CLI, server, analysis, and distribution

Use this reference for command behavior, automation output, editor integration,
dependency graphs, file discovery, containers, binaries, and source builds.

## Formatting commands and exit status

`ruff format --exit-non-zero-on-format` writes formatting changes but exits
nonzero if it modified any files (0.11.0):

```console
ruff format --exit-non-zero-on-format .
```

In 0.16.0-guide, `ruff format --check` accepts the linter's output formats,
including CI annotation formats:

```console
ruff format --check --output-format github .
```

Both `ruff check` and `ruff format --check` now include fix diffs in their
human-readable output. Wrappers that parse logs need to tolerate the added
diffs (0.16.0-guide).

## Watch mode and structured output

`ruff check --watch` respects `--output-format` and defaults to `full`
(0.15.0).

JSON consumers must accept null for `filename`, `location`, `end_location`,
`fix.edits[].location`, and `fix.edits[].end_location`. These fields no longer
always use empty strings or row-1/column-1 placeholders (0.16.0-guide).

## Language server

Server logging is controlled solely by `logLevel`, whose default is `info`.
The LSP `trace` value no longer enables or disables logging. Code-action
requests ignore diagnostics produced by other sources (0.9.0).

`ruff.printDebugInformation` no longer produces logging output (0.10.0).

The server can use `uv` as an alternative formatter backend (0.13.0).

Preview output, LSP hovers, and code actions prefer human-readable rule names
in 0.15.0. `ruff rule` accepts those names, while unknown selectors warn rather
than fail.

The server supports formatting labeled Python code blocks in Markdown. Its file
handling follows configured extension mappings, including an explicit mapping
such as this for Quarto files (0.15.0):

```toml
[tool.ruff]
extension = { qmd = "markdown" }
```

## Dependency-graph analysis

`ruff analyze graph` accepts a virtual environment so dependency analysis can
resolve imports installed there (0.11.0).

Dependency analysis can skip imports in `TYPE_CHECKING` blocks, operate on
Jupyter notebooks, and use configured `src` directories when resolving imports
(0.14.0).

Because the isort implementation checks a module's full path against the
configured project root or roots, nested modules can change first-party
classification and import section (0.11.0).

## Input discovery and executable files

Preview discovery includes `*.pyw` by default (0.14.0).

Markdown discovery became a preview default in 0.15.5, and Markdown formatting
became default behavior in 0.16.0-guide. Unlabeled code fences are not treated
as Python by the introduced Markdown formatter; labeled Python, `pycon`, and
Quarto markers are supported. Configured extensions participate in discovery,
code-block language selection, and server handling.

`EXE003` accepts `uv run` shebangs (0.11.0).

## Container images

The `ruff:alpine` image moved from Alpine 3.20 to 3.21 in 0.10.0-guide. The
`ruff:alpine3.20` tag is no longer updated.

By 0.15.0, `ruff:alpine` uses Alpine 3.23. `ruff:debian` and
`ruff:debian-slim` use Debian 13 “Trixie.” Pin an explicit image tag and base
variant when operating-system packages matter.

## Binaries, source distributions, and release artifacts

Source distributions stopped including `rust-toolchain.toml` in 0.12.0.
Downstream packagers can build with a toolchain compatible with Ruff's minimum
supported Rust version rather than the higher release-build toolchain.

Version 0.14.12 was published to PyPI without a GitHub release or tag because
of a WASM publishing issue. Version 0.14.13 has identical contents and serves
as the follow-up release (0.14.0).

In 0.15.0:

- source builds require Rust 1.91 or newer;
- release binaries no longer include big-endian `ppc64`; and
- WASM artifacts are no longer attached to GitHub releases.

Account for those changes in downstream packaging, architecture matrices, and
artifact download scripts.
