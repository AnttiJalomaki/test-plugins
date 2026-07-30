# Semantic Metadata and Artifacts

Use this reference when authoring Semantic Layer or OSI YAML, adding metadata
to resources, or consuming generated artifacts and structured logs. Relevant
extraction sections: 1.9.0, 1.10.0, and 1.12.0.

## Semantic manifest and query features

Core 1.9 semantic manifests can represent:

- cumulative type parameters;
- metric `time_granularity`;
- sub-daily granularities.

Time-spine YAML supports expanded time-spine settings and uniquely named
`custom_granularities`. Saved queries accept `order_by` and `limit`.

Core 1.10 adds tags to saved queries. Offset windows support custom grains.

## Semantic metadata config

Core 1.10 expands metadata support:

- groups accept `description` and `config.meta`;
- exposures accept tags and meta config;
- Semantic Layer dimensions, measures, and entities accept meta config;
- column config meta and tags propagate to tests on that column.

Keep metadata in `config` where the resource schema expects it rather than
relying on unsupported top-level custom properties.

## V2 Semantic Layer YAML

Core 1.12 parses new-style V2 Semantic Layer YAML for:

- standalone and model-attached metrics;
- entities and derived entities;
- derived dimensions;
- `agg_time_dimension`;
- object-style Semantic Model config;
- `primary_entity`.

Model-as-Semantic-Model and column-dimension parsing are explicitly not fully
ready in this release. Treat these two surfaces as incomplete even though
other V2 forms parse.

## OSI documents

Core reads OSI documents from `OSI/` or `osi/` into the manifest. The OSI
directory is configurable. Parsing also writes an OSI document, so tooling
that watches the project or target outputs should account for this generated
file.

## Macro and analysis properties

Macro property entries accept `config` containing `meta` and `docs`:

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

## Artifact additions

Core 1.10 adds:

- an invocation-start timestamp and quoting configuration to artifact
  metadata;
- `doc_blocks` to manifest nodes and columns;
- `node_checksum` to structured-log `node_info`;
- `name` to package lock entries;
- support for uploading artifacts to dbt Cloud.

Consumers should tolerate these additive fields and should use the invocation
start timestamp when correlating artifacts with logs.

## Compilation and tooling outputs

Core 1.12 changes several integration surfaces:

- `dbt compile` writes compiled snapshot SQL beneath `target/compiled/`.
- The Jinja `graph` includes unit tests.
- Python-model parsing recognizes `config.meta_get`.
- `NodeStatus` and `RunStatus` add `Reused`.
- Model records from `dbt ls --output json` gain runtime-only
  `direct_parents`, listing the nearest public ancestors.

`direct_parents` does not change `manifest.json`; do not expect it in stored
manifest nodes merely because it appears in `dbt ls` output.
