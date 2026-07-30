# Queries, Observability, and Terminal UI

## Terminal UI controls

Expect selection, task-list visibility, and pinning state to persist between
invocations (since 2.4.0). Use these controls:

| Key | Action |
| --- | --- |
| `h` | Toggle the task list |
| `c` | Copy highlighted logs |
| `j` / `k` | Select tasks |
| `p` | Pin or unpin a task |
| `u` / `d` | Scroll logs |
| `m` | List all keybindings |

Press `/` to enter a task search query (since 2.6.0). The task list filters so
only matching tasks are selected.

## Graph and dry-run inspection

Use JSON output from `turbo ls` to inspect package dependents (since 2.6.0).
Dry-run and summary output also expose `with` sidecar relationships.

Run `turbo devtools` for Package Graph and Task Graph views that hot-reload as
the repository changes (since 2.7.0). Inspect direct and transitive
relationships when diagnosing cache misses:

```bash
turbo devtools
```

For graph files, replace deprecated `.png`, `.jpg`, or `.pdf` output with
`.svg`, `.html`, `.mermaid`, or `.dot` (deprecated forms documented in 2.9.0).
Replace `.json` graph output with `turbo query`.

## Stable repository queries

Use `turbo query` as the stable repository-query interface (since 2.9.0). With
no query, it opens GraphiQL. Supply GraphQL inline or with `--file`, and inspect
the schema with `--schema`:

```bash
turbo query
turbo query --schema
turbo query '{ packages { items { name } } }'
turbo query --file=query.gql
```

Use the `affected` shorthand for structured JSON describing changed tasks or
packages. Use `ls` for pretty package details, JSON output, affected-only
results, and selectors:

```bash
turbo query affected --tasks build
turbo query affected --packages
turbo query ls web --output=json
turbo query ls --affected --filter='./apps/*'
```

Replace deprecated `turbo-ignore` with `turbo query affected`.

## Intersect affected and filtered scopes

Combine `--affected` and `--filter` to select only tasks or packages that match
both constraints (since 2.10.0). Negate a filter to remove packages from the
affected set:

```bash
turbo run build --affected --filter=web
turbo run build --affected --filter=!docs
turbo query ls --affected --filter=my-app
```

## OpenTelemetry metrics

Enable the `experimentalObservability` Future Flag and configure an OTLP
endpoint to export experimental metrics (since 2.9.0), including
`turbo.run.duration_ms`, `turbo.run.tasks.cached`, and
`turbo.run.tasks.failed`:

```json
{
  "futureFlags": { "experimentalObservability": true },
  "experimentalObservability": {
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector.example.com:4317",
      "protocol": "grpc"
    }
  }
}
```

## Structured logs

Use `--json` to stream experimental NDJSON objects containing `timestamp`,
`source`, `level`, and `text` (since 2.9.0). Use `--log-file` to preserve normal
terminal output while writing structured logs to
`.turbo/logs/<epoch-millis>.json`; provide a value for a custom path. Combine
the flags when both stream and file output are required:

```bash
turbo run build --json
turbo run build --log-file
turbo run lint --json --log-file=logs.json
```

## Profiles

Omit filenames from `--profile` and `--anon-profile` when the default output is
sufficient (since 2.9.0). Each profile also produces a Markdown companion next
to its trace:

```bash
turbo run build --profile
turbo run build --anon-profile
```
