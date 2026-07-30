---
name: turborepo-knowledge-patch
description: Turborepo
version: 2.10.0
license: MIT
metadata:
  author: Nevaberry
---



# Turborepo Knowledge Patch

Use this skill when configuring, upgrading, debugging, or operating a
Turborepo workspace. It is especially useful for task-graph behavior, cache
correctness, package boundaries, repository queries, and migration away from
interfaces that are being removed.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-and-configuration.md](references/migration-and-configuration.md) | Configuration formats and inheritance, schema selection, future flags, lint integrations, deprecations, and codemods |
| [task-execution-and-inputs.md](references/task-execution-and-inputs.md) | Sidecars, continuation, task and package cycles, watch behavior, structured inputs, hashing, and graceful shutdown |
| [caching-environment-and-package-managers.md](references/caching-environment-and-package-managers.md) | Environment handling, cache precedence and eviction, worktree sharing, Bun, Yarn catalogs, and pruning |
| [repository-analysis-and-structure.md](references/repository-analysis-and-structure.md) | Package boundaries, repository queries, affected scopes, microfrontends, graph output, catalogs, and Cargo workspaces |
| [developer-tools-and-observability.md](references/developer-tools-and-observability.md) | Terminal UI, graph devtools, documentation access, structured logs, metrics, profiling, and the packaged guidance skill |

## Migrate deprecated interfaces first

When preparing a repository for the next major release, replace deprecated
commands, flags, environment variables, and configuration instead of building
new automation around them.

| Deprecated interface | Replacement |
| --- | --- |
| `turbo-ignore` | `turbo query affected` |
| `turbo scan` | Remove it; there is no replacement |
| `--parallel` | Task-level `persistent` and `with` |
| `--no-cache` | `--cache=local:r,remote:r` |
| `TURBO_REMOTE_ONLY`, `--remote-only` | `--cache=remote:rw` |
| `TURBO_REMOTE_CACHE_READ_ONLY`, `--remote-cache-read-only` | `--cache=local:rw,remote:r` |
| `turbo prune --scope web` | `turbo prune web` |
| JSON `--graph` output | `turbo query` |
| PNG, JPG, or PDF graph output | SVG, HTML, Mermaid, or DOT |

`turbo run` and `turbo watch` no longer use the daemon. Remove
`TURBO_DAEMON`, `--daemon`, `--no-daemon`, and the `daemon` configuration key.

Run the migration codemod after reviewing these replacements:

```bash
npx @turbo/codemod migrate
```

The codemod understands package-manager catalogs.

## Choose the right configuration form

Use the schema shipped with the installed package so editor validation follows
the local Turborepo version:

```json
{
  "$schema": "./node_modules/turbo/schema.json"
}
```

Adjust the relative path in package-level configuration. Both `turbo.jsonc`
and trailing commas in `turbo.json` are supported.

Package configuration may extend the root with `//` and another workspace
package by name:

```json
{
  "extends": ["//", "web"]
}
```

Local arrays replace inherited arrays unless they include
`$TURBO_EXTENDS$`:

```json
{
  "extends": ["//"],
  "tasks": {
    "build": {
      "outputs": ["$TURBO_EXTENDS$", ".next/**"]
    }
  }
}
```

Task `description` values are informational and never change execution.

## Model long-running work explicitly

Use `with` when a persistent task needs another long-running task to coexist
with it. Use `dependsOn` only for completion ordering.

```json
{
  "tasks": {
    "dev": {
      "with": ["api#start"],
      "persistent": true,
      "cache": false
    }
  }
}
```

To continue independent work after failures without running dependents of
failed tasks:

```bash
turbo run test --continue=dependencies-successful
```

Package Graph cycles are not automatically fatal. The Task Graph is decisive:
a simple task may run in a cyclic package graph, while `^build` still fails
when it creates a cyclic task dependency.

## Make task inputs precise

Anchor repository-level files with `$TURBO_ROOT$` rather than package-relative
`../` traversal:

```json
{
  "tasks": {
    "build": {
      "inputs": ["$TURBO_ROOT$/important-file.txt"]
    }
  }
}
```

For generated files, use a structured `inputs` entry with `mode: "jit"` so
that portion of the hash is computed after dependencies finish. For an
upstream task's stable public artifacts, use `mode: "dependencyOutputs"` with
`globs` and `from`. This keeps downstream tasks cached when implementation
changes do not alter the selected outputs.

Do not use deferred inputs as a substitute for dependency edges. The producing
task must still appear in `dependsOn`.

## Query before running broad scopes

Use stable repository queries for scripts and diagnostics:

```bash
turbo query affected --tasks build
turbo query affected --packages
turbo query ls web --output=json
turbo query ls --affected --filter='./apps/*'
```

`turbo query` accepts inline GraphQL or `--file`, exposes its schema with
`--schema`, and opens GraphiQL when called without a query.

`--affected` and `--filter` intersect. Use this to narrow changed work or to
exclude a package:

```bash
turbo run build --affected --filter=web
turbo run build --affected --filter=!docs
```

When opting into task-aware future behavior, remember that relevant future
flags alter selection semantics and every enabled future flag affects the
global hash.

## Protect cache correctness

Environment-variable declarations are part of cache correctness. ESLint Flat
Config can register `eslint-plugin-turbo` directly or spread the shareable
configuration. Biome can also report undeclared environment variables, but
its rule must be enabled explicitly.

`DISPLAY` is passed through by default. Negated `passThroughEnv` entries can
exclude built-ins and values inherited from `globalPassThroughEnv`. Force mode
takes precedence over other cache settings, including `remoteCache.enable`.

Use `cacheMaxAge` and `cacheMaxSize` to opt into local artifact eviction:

```json
{
  "cacheMaxAge": "7d",
  "cacheMaxSize": "10GB"
}
```

At the start of `turbo run`, eviction removes expired entries and then the
oldest entries needed to satisfy the size cap. It runs in a background thread.
Linked Git worktrees share their local cache automatically.

## Respect package-manager granularity

Bun repositories can be pruned when using Bun 1.2 or newer. The text
`bun.lock` v1 format is parsed granularly, so a dependency edit invalidates
only affected packages. Yarn catalogs receive similarly granular handling
with Yarn 4.10.0 or newer.

To honor ignored files during pruning:

```bash
turbo prune web --use-gitignore
```

## Inspect boundaries and graphs

Run `turbo boundaries` to find imports outside a package, undeclared package
dependencies, circular package dependencies, and dynamic imports. Boundary
analysis also understands package rules, implicit dependencies, and
TypeScript path aliases.

Use `turbo devtools` for hot-reloading Package Graph and Task Graph views.
Use `turbo query` or `turbo ls` when a machine-readable answer is preferable.
Cycle diagnostics can suggest complete sets of dependency edges where removing
any one set breaks the package cycle.

## Observe and debug execution

Use `--json` for NDJSON logs and `--log-file` to preserve normal terminal
output while writing structured logs. The options can be combined.

Enable experimental observability through both the corresponding future flag
and the OTLP configuration before expecting metrics. For local profiling,
`--profile` and `--anon-profile` may omit the filename and produce a Markdown
companion beside the trace.

The terminal UI persists selection, list visibility, and pinned tasks. Press
`m` to see all keybindings and `/` to search tasks.

On `SIGINT` or `SIGTERM`, Turborepo forwards the signal and waits for task
cleanup. A second `Ctrl+C` forces immediate termination.
