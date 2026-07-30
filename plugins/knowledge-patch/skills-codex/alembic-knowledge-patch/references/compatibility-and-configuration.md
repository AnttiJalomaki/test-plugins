# Compatibility and Configuration

## Runtime requirements

Alembic 1.15.0 drops Python 3.8 and SQLAlchemy 1.3. It requires Python 3.9 or
newer and SQLAlchemy 1.4 or newer.

Alembic 1.17.0 raises the Python minimum to 3.10; Python 3.9 is no longer
supported.

Alembic 1.18.0 raises the SQLAlchemy minimum from 1.4.0 to 1.4.23.

## Yanked 1.15.0 package

The PEP 621 packaging change omitted Alembic's template files from the 1.15.0
wheel, so that release was yanked. The corrected wheel shipped in 1.15.1.
Install 1.15.1 or later when using that series.

## Source-build requirement

As of 1.16.0, building Alembic from source requires setuptools 77.0.3 or newer.
This accompanies the move to PEP 639 license metadata.

## Split source settings from deployment settings

As of 1.16.0, source-code and migration-generation settings can live in
`pyproject.toml`. These include local paths and post-write hooks.

Use TOML lists for `version_locations` and `prepend_sys_path` to avoid string
separator ambiguity. In TOML configuration, `%(here)s` resolves relative to
the parent directory of the TOML file.

Database connectivity and logging remain deployment concerns. Supply them
through `alembic.ini` or `env.py`. If `env.py` supplies both, the `pyproject`
initialization template permits a project without `alembic.ini`.

## Use cross-platform INI path splitting

`path_separator` supersedes `version_path_separator` and governs both
`version_locations` and `prepend_sys_path`. The value `os` uses
`os.pathsep`, making the configuration portable:

```ini
[alembic]
path_separator = os
```

The setting defaults to `os` in newly configured environments. A configuration
that omits it keeps the older splitting behavior and emits a deprecation
warning.

## Pass path-like objects through public APIs

Since 1.16.0, public command, configuration, and script APIs that accept string
paths also accept `os.PathLike` objects. Public accessors that return paths
continue to return strings.

Private underscored APIs are not covered by that return-type guarantee. After
the path-handling refactor, they may return `pathlib.Path` objects.
