# Project Configuration and Semantic Layer

## Column, source, and resource metadata

Columns accept a `config` mapping, and source freshness can be nested under
`config` with explicit null values preserved (from `1.9.0`):

```yaml
models:
  - name: orders
    columns:
      - name: id
        config:
          meta:
            owner: analytics
```

Core 1.10.0 expands metadata support:

- saved queries accept `tags`;
- groups accept `description` and `config.meta`;
- exposures accept tags and meta config;
- Semantic Layer dimensions, measures, and entities accept meta config;
- column `meta` and tags propagate to tests;
- offset windows accept custom grains.

Core 1.11.0 adds model-Jinja metadata accessors. Use `config.meta_get(key)` for
an optional value and `config.meta_require(key)` for a required value:

```jinja
{{ config(meta={"owner": "finance", "policy": "restricted"}) }}
{% set owner = config.meta_get("owner") %}
{% set policy = config.meta_require("policy") %}
```

Python-model parsing recognizes `config.meta_get` in Core 1.12.0.

Macro properties also accept `config.meta` and `config.docs` in Core 1.12.0:

```yaml
macros:
  - name: cents_to_dollars
    config:
      meta: {owner: finance}
      docs: {show: true}
```

Analyses can be enabled or disabled in `dbt_project.yml` at project or folder
scope:

```yaml
analyses:
  my_project:
    staging:
      +enabled: false
```

## Semantic Layer evolution

The `1.9.0` batch adds cumulative type parameters, metric
`time_granularity`, and sub-daily granularities to semantic manifests. Time
spine YAML gains new settings and uniquely named `custom_granularities`; saved
queries accept `order_by` and `limit`.

Core 1.12.0 parses new-style V2 Semantic Layer YAML for:

- standalone and model-attached metrics;
- entities and derived entities;
- derived dimensions and `agg_time_dimension`;
- object-style Semantic Model config;
- `primary_entity`.

Model-as-Semantic-Model and column-dimension parsing are explicitly not fully
ready in this release. OSI documents are read from `OSI/` or `osi/` into the
manifest, the directory is configurable, and dbt writes an OSI document after
parsing.

## Catalog integration

Core 1.10.0 parses `catalogs.yml`; from 1.10.12, it also parses the file during
`parse`, `seed`, and `test`. Catalog integration config accepts `file_format`.

Core 1.12.0 makes Catalog Configuration V2 opt-in:

```yaml
flags:
  use_catalogs_v2: true
```

When enabled, every command that requires a manifest loads catalog config. For
V1 integrations, `catalog_database` can override the database name for any
catalog type and has highest priority during database-name generation.

## Additional project inputs

Core 1.12.0 supports:

- project variables declared in `vars.yml`;
- environment variables automatically loaded from `.env`;
- SQL and Markdown files with Jinja-suffixed extensions such as `.sql.jinja`
  and `.md.jinja`.

Keep `.env` out of source control when it contains secrets.

The `MAXIMUM_SEED_SIZE_MIB` seed limit is configurable. The `--sqlparse`
option configures parser limits without requiring a pinned `sqlparse` release.

## Private packages and protected refs

Core 1.12.0 supports private Git packages in both `packages.yml` and
`dependencies.yml`. dbt resolves their URLs from a configured environment
variable when present; otherwise it constructs an SSH URL.

Macros invoked with `run-operation` can call `ref()` on private and protected
models. Ad-hoc SQL does not need a wrapper macro:

```bash
dbt run-operation --sql 'select count(*) from {{ ref("orders") }}'
```
