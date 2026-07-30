# Analysis, Discovery, and Distribution

## Import classification and dependency graphs

Ruff's isort implementation checks a module's full path against the configured
project root or roots when classifying it as first-party. Nested modules may
move between import sections (0.11.0).

`ruff analyze graph` accepts a virtual environment so dependency analysis can
resolve imports installed there (0.11.0).

Dependency analysis can skip imports inside `TYPE_CHECKING`. Graphs also work
with Jupyter notebooks and use configured `src` directories for import
resolution (0.14.0).

## File discovery

Preview mode includes `*.pyw` files by default (0.14.0).

Preview formatting discovers Markdown by default from 0.15.5. `.qmd` no longer
has an implicit special case; map it to `markdown` explicitly. Configured
extensions participate in discovery, Markdown code-block language selection,
and later server handling (0.15.0).

Markdown formatting is enabled by default in 0.16, rather than only in preview
(0.16.0-guide).

## Containers

The `ruff:alpine` image moved from Alpine 3.20 to Alpine 3.21, and
`ruff:alpine3.20` stopped receiving updates (0.10.0-guide).

The `ruff:alpine` image later moved to Alpine 3.23. `ruff:debian` and
`ruff:debian-slim` use Debian 13 “Trixie” (0.15.0).

## Release artifacts and source builds

Source distributions no longer include `rust-toolchain.toml`. Downstream
packagers can build with a compiler compatible with Ruff's minimum supported
Rust version instead of the higher release-build toolchain (0.12.0).

Version 0.14.12 was published to PyPI but has no corresponding release or tag
because of a WASM publishing issue. Version 0.14.13 has identical contents and
is the follow-up release (0.14.0).

Release binaries no longer include big-endian `ppc64`. WASM artifacts are no
longer attached to releases. Source builds require Rust 1.91 or newer
(0.15.0).
