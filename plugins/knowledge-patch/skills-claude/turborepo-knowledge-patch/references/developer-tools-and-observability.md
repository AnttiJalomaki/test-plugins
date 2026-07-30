# Developer tools and observability

Use this reference for terminal navigation, live graph inspection,
documentation discovery, structured logs, OpenTelemetry metrics, and
profiling.

## Persistent terminal UI state and controls

The terminal UI remembers the selected task, task-list visibility, and task
pinning between invocations (since 2.4.0).

| Key | Action |
| --- | --- |
| `h` | Toggle the task list |
| `c` | Copy highlighted logs |
| `j` / `k` | Select tasks |
| `p` | Pin or unpin a task |
| `u` / `d` | Scroll logs |
| `m` | List all keybindings |

Press `/` to enter a search query (since 2.6.0). The task list is filtered so
only matching tasks are selected.

## Live graph devtools

`turbo devtools` provides hot-reloading Package Graph and Task Graph views
(since 2.7.0):

```bash
turbo devtools
```

The views expose direct and transitive relationships, which can explain task
selection and cache misses as the repository changes.

Dry-run and summary output show `with` sidecar relationships (since 2.6.0).
JSON output from `turbo ls` includes package dependents.

## Documentation from HTTP and the CLI

Documentation routes return Markdown when requested with
`Accept: text/markdown`, and appending `.md` to a route also returns Markdown
(since 2.8.0). `/sitemap.md` is a machine-readable index. Version-pinned
documentation is available from version subdomains such as
`v2-7-6.turborepo.dev`.

```bash
curl -sL -H "Accept: text/markdown" https://turborepo.dev/repo/docs
curl -sL https://turborepo.dev/sitemap.md
```

Search documentation directly from the terminal:

```bash
turbo docs "package configurations"
```

The official Turborepo Agent Skill packages Turborepo and monorepo patterns
and anti-patterns for compatible coding assistants (since 2.8.0):

```bash
npx skills add vercel/turborepo
```

## Structured logging

`--json` streams newline-delimited JSON objects with `timestamp`, `source`,
`level`, and `text` fields (since 2.9.0):

```bash
turbo run build --json
```

`--log-file` keeps normal terminal output while writing structured logs. With
no path it writes `.turbo/logs/<epoch-millis>.json`; it also accepts a custom
path and can be combined with `--json`:

```bash
turbo run build --log-file
turbo run lint --json --log-file=logs.json
```

Choose `--json` for a structured stdout consumer and `--log-file` when people
still need the ordinary terminal display.

## Experimental OpenTelemetry metrics

Enable the `experimentalObservability` future flag and configure an OTLP
endpoint before expecting metric export (since 2.9.0):

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

Exported metrics include:

- `turbo.run.duration_ms`
- `turbo.run.tasks.cached`
- `turbo.run.tasks.failed`

The future flag changes the global hash. Confirm that both the flag and
`otel.enabled` are set when the collector receives nothing.

## Profiling

`--profile` and `--anon-profile` no longer require a filename (since 2.9.0):

```bash
turbo run build --profile
turbo run build --anon-profile
```

Each profile also produces a Markdown companion alongside the trace.

## Shutdown behavior while observing tasks

On `SIGINT` or `SIGTERM`, Turborepo forwards the signal and waits for task
cleanup handlers (since 2.10.0). A second `Ctrl+C` forces immediate exit. Give
the first signal time to flush logs, traces, or other shutdown output before
forcing termination.
