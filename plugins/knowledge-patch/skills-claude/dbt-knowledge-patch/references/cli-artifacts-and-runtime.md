# CLI, Artifacts, and Runtime

## Quiet and machine-readable output

In the `1.9.0` batch, `dbt show` and `dbt compile` retain parseable JSON or text
results with `--quiet`. Automation can suppress event logs without discarding
the command result.

Core 1.11.0 lets `dbt ls --output json --output-keys` select nested paths:

```bash
dbt ls --output json --output-keys name config.materialized
```

## Docs serving and working directory

`dbt docs serve` accepts `--host` and defaults to `127.0.0.1`. Bind to all
interfaces only when remote reachability is deliberate:

```bash
dbt docs serve --host 0.0.0.0
```

`dbt deps`, `dbt clean`, and `dbt init` no longer change the process working
directory. Embedded callers that depended on that side effect must manage paths
explicitly.

## Catalogs, artifacts, logs, and lock files

Core 1.10.0 parses `catalogs.yml`; from 1.10.12, `parse`, `seed`, and `test` do
so as well. Catalog integration accepts `file_format`. Catalog V2 details are in
[project-config-and-semantic-layer.md](project-config-and-semantic-layer.md).

Core 1.10.0 also adds:

- an invocation-start timestamp and quoting config to artifact metadata;
- `doc_blocks` to manifest nodes and columns;
- `node_checksum` to structured-log `node_info`;
- `name` to package lock entries;
- support for uploading artifacts to dbt Cloud.

Core 1.12.0 writes compiled snapshot SQL under `target/compiled/`. `NodeStatus`
and `RunStatus` add `Reused`. Model records from `dbt ls --output json` include
a runtime-only `direct_parents` field containing the nearest public ancestors;
this does not change `manifest.json`.

## External V2 parser

Core 1.12.0 adds `--use-v2-parser`, which bypasses Core's parser, runs an
external parser, and loads the resulting `manifest.json` back into the runtime
manifest. Choose the command with `--v2-parser` or
`DBT_ENGINE_V2_PARSER`; the default is
`dbt-core-experimental-parser parse`. It may also be configured under project
`flags` and requires `dbt-core-experimental-parser>=2.0.0a4`.

```bash
dbt parse --use-v2-parser \
  --v2-parser "dbt-core-experimental-parser parse"
```

## Ad-hoc SQL and input limits

Core 1.12.0 lets `dbt run-operation --sql` execute ad-hoc SQL or Jinja without
a wrapper macro:

```bash
dbt run-operation --sql 'select count(*) from {{ ref("orders") }}'
```

`MAXIMUM_SEED_SIZE_MIB` makes the seed-size ceiling configurable. The
`--sqlparse` option configures SQL parser limits instead of requiring a pinned
`sqlparse` release.

## Runtime compatibility

The 1.9 line drops Python 3.8 and raises the minimum `dbt-adapters` version to
1.9.0.

Core 1.10.0 adds Python 3.13 support and allows Pydantic v1 or v2. Patch
releases in this line:

- require JSON Schema 4.19.1 or newer;
- move to Protobuf 6;
- cap `sqlparse` below 0.5.5;
- require `dbt-common>=1.37.3`;
- set the `dbt-adapters` lower bound to 1.16.5 from 1.10.10;
- temporarily cap `dbt-adapters` below 1.24 in 1.10.21, then restore the upper
  bound to below 2.0 in 1.10.22.

Core 1.11.0 drops Python 3.9, requiring Python 3.10 or newer.

Core 1.12.0 supports Python 3.14 and raises minimum dependencies to Click
8.3.0, `dbt-common` 1.37.3, and `dbt-adapters` 1.24.5.
