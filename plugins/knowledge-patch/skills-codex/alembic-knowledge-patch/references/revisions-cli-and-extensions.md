# Revisions, CLI, and Extensions

## Register custom commands publicly

Since 1.16.0, applications that extend Alembic's command-line tool can use the
public registration API:

```python
command_line.register_command(my_command)
```

`CommandLine.register_command()` replaces reliance on the formerly internal
registration mechanism.

## Avoid colons in revision identifiers

As of 1.17.0, custom revision IDs cannot contain `:` because the character is
reserved for revision range syntax. Use an identifier such as `REV_1`, not
`REV:1`.

## Verify that every head is applied

Since 1.17.0, `command.current()` accepts `check_heads=True`. The equivalent
CLI option is:

```console
alembic current --check-heads
```

The check raises `DatabaseNotAtHead` through the Python API, or exits nonzero
through the CLI, if any head revision has not been applied.

## Replace an operation implementation

Since 1.17.0, an extension can pass `replace=True` to
`Operations.implementation_for()` to replace an existing implementation:

```python
@Operations.implementation_for(CreateTableOp, replace=True)
def create_table(operations, operation):
    ...
```

This supports overriding built-in operations such as `CreateTableOp`, not only
registering implementations for new operation types.

## Place revisions in date directories

As of 1.18.0, `file_template` accepts directory separators and creates the
necessary directories:

```toml
[tool.alembic]
file_template = "%(year)d/%(month).2d/%(day).2d_%(rev)s_%(slug)s"
recursive_version_locations = true
```

Set `recursive_version_locations = true`; otherwise later commands do not find
the revision files nested below the version location.

## Build automatically loaded plugins

The 1.18.0 `Plugin` interface lets third-party extensions load automatically
and register operations, implementations, and autogenerate comparators.
Register a comparator through `Plugin.add_autogenerate_comparator()`.

Select built-in and third-party comparison plugins for an environment with:

```python
EnvironmentContext.configure(autogenerate_plugins=[...])
```

Existing add-ons continue to work without adopting plugin entry points.

## Splice a merge revision

Since 1.18.0, merge revisions can be created from non-head revisions by passing
`--splice`:

```console
alembic merge --splice rev_a rev_b
```

The programmatic `command.merge()` API accepts the corresponding `splice`
parameter.

## Resolve effective heads with dependencies

As of 1.18.0, request dependency-aware head lookup with:

```python
heads = script.get_heads(consider_depends_on=True)
```

`ScriptDirectory.get_heads(consider_depends_on=True)` excludes a nominal head
that another revision references through `depends_on`. The result matches the
effective heads stored in `alembic_version` after all upgrades.
