# Revisions, CLI, and extensions

## CLI extension and validation

### Public custom-command registration

`CommandLine.register_command()` is the public registration mechanism for
applications that extend the Alembic command-line tool. Use it instead of the
previously internal mechanism. (since 1.16.0)

```python
command_line.register_command(my_command)
```

### Verify that every head is applied

`command.current()` accepts `check_heads=True`. It raises
`DatabaseNotAtHead` when any head revision is not applied. The matching CLI
option exits nonzero in that situation, making it suitable for deployment or
health checks. (since 1.17.0)

```console
alembic current --check-heads
```

## Revision identifiers and file layout

### Colons are reserved

Custom revision identifiers cannot contain `:` because the character is
reserved for revision-range syntax. Use an identifier such as `REV_1` rather
than `REV:1`. (since 1.17.0)

### Date-organized revision paths

`file_template` accepts directory separators and creates the required
directories automatically. When using nested paths, also enable recursive
version locations; otherwise later commands will not discover the generated
revision files. (since 1.18.0)

```toml
[tool.alembic]
file_template = "%(year)d/%(month).2d/%(day).2d_%(rev)s_%(slug)s"
recursive_version_locations = true
```

## Merge and head resolution

### Splice merge revisions

The merge command accepts `--splice`, allowing a merge revision to be created
from revisions that are not heads. Programmatic callers use the corresponding
`splice` parameter on `command.merge()`. (since 1.18.0)

```console
alembic merge --splice rev_a rev_b
```

### Dependency-aware heads

`ScriptDirectory.get_heads(consider_depends_on=True)` removes nominal heads
that serve as a `depends_on` dependency of another revision. The result matches
the effective heads recorded in `alembic_version` after all upgrades. (since
1.18.0)

```python
heads = script.get_heads(consider_depends_on=True)
```

## Extension and autogenerate plugins

The `Plugin` interface supports automatically loaded extensions that register:

- operations;
- operation implementations;
- autogenerate comparators through `Plugin.add_autogenerate_comparator()`.

Built-in and extension comparison plugins can be selected for each environment
with `EnvironmentContext.configure(autogenerate_plugins=...)`. Existing add-ons
continue to work without adopting plugin entry points. (since 1.18.0)
