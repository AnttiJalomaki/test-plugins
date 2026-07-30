# Compatibility and configuration

## Runtime and packaging requirements

### Python and SQLAlchemy floors

Alembic 1.15.0 requires Python 3.9 or newer and SQLAlchemy 1.4 or newer. It
drops Python 3.8 and SQLAlchemy 1.3. Check application, migration runner, CI,
and image pins together because migrations can execute in an environment
separate from the application. (since 1.15.0)

Alembic 1.17.0 requires Python 3.10 or newer; Python 3.9 is no longer
supported. (since 1.17.0)

Alembic 1.18.0 requires SQLAlchemy 1.4.23 or newer, raising the earlier
SQLAlchemy 1.4.0 floor. (since 1.18.0)

### Yanked 1.15.0 wheel

The move to PEP 621 packaging omitted Alembic's template files from the 1.15.0
wheel, so the release was yanked. Install 1.15.1 or later in that series; the
wheel was corrected in 1.15.1. (1.15.0)

### Source builds

The source-build requirement for setuptools is 77.0.3 or newer following the
adoption of PEP 639 license metadata. Treat this as a build-tool requirement,
not an Alembic runtime dependency. (since 1.16.0)

## Source configuration in `pyproject.toml`

Alembic can keep source-code and generation settings in `pyproject.toml`,
including:

- local paths;
- post-write hooks;
- `version_locations`;
- `prepend_sys_path`.

TOML lists remove separator ambiguity for `version_locations` and
`prepend_sys_path`. In TOML configuration, `%(here)s` resolves relative to the
parent directory of the TOML file. (since 1.16.0)

Database connectivity and logging remain deployment settings. Supply them
through `alembic.ini` or `env.py`. If `env.py` handles those settings, use the
`pyproject` initialization template when an `alembic.ini` file is unnecessary.

## Cross-platform path separation

`path_separator` supersedes `version_path_separator` and applies to both
`version_locations` and `prepend_sys_path`. Set it to `os` to split values with
`os.pathsep`: (since 1.16.0)

```ini
[alembic]
path_separator = os
```

New configurations default to `os`. Configurations that omit the setting keep
the older splitting behavior and emit a deprecation warning, so add the setting
explicitly when updating an existing configuration.

TOML lists are preferable when the values live in `pyproject.toml`; no string
separator then needs to be inferred.

## Public path API contract

Public command, configuration, and script APIs that accept string paths also
accept `os.PathLike` objects. Public accessors that return paths continue to
return strings. (since 1.16.0)

Private underscored APIs are outside that return-type promise and may return
`pathlib.Path` objects after the path-handling refactor. Extensions should use
public APIs or normalize private return values if private access is unavoidable.
