---
name: turborepo-knowledge-patch
description: Turborepo
version: 2.10.0
license: MIT
metadata:
  author: Nevaberry
---


# Turborepo Knowledge Patch

Inspect the repository's `package.json`, lockfile, and installed `turbo` version
before changing configuration or commands. Apply version-attributed guidance only
when the installed version includes that behavior. Prefer the repository's code,
configuration, and test results when they disagree with this guide.

## Reference index

| Reference | Topics |
| --- | --- |
| [configuration-and-inputs.md](references/configuration-and-inputs.md) | Local schemas, JSONC, environment handling, inheritance, task descriptions, Future Flags, and structured inputs |
| [execution-and-caching.md](references/execution-and-caching.md) | Sidecars, continuation, Watch Mode, package-manager invalidation, pruning, worktree caches, shutdown, and eviction |
| [queries-observability-and-ui.md](references/queries-observability-and-ui.md) | Stable queries, affected selection, graph tools, terminal UI, structured logs, OpenTelemetry, and profiling |
| [boundaries-and-linting.md](references/boundaries-and-linting.md) | Package-boundary checks, cycle diagnostics, ESLint Flat Config, and Biome environment linting |
| [ecosystem-and-migrations.md](references/ecosystem-and-migrations.md) | Microfrontends, documentation tooling, migration deprecations, catalogs, Cargo workspaces, and agent guidance |

## Apply this patch

1. Determine the installed Turborepo and package-manager versions.
2. Read the reference matching the task before editing commands or configuration.
3. Check inline version markers before relying on newer syntax or behavior.
4. Treat Future Flags and experimental interfaces as opt-in behavior.
5. Validate changes with a dry run, query, or focused task execution.

## Breaking changes and deprecations

### Replace daemon controls

Do not design automation around the Turborepo daemon. `turbo run` and
`turbo watch` execute without it. Remove these deprecated controls when
migrating configuration or scripts:

- `TURBO_DAEMON`
- `--daemon`
- `--no-daemon`
- the `daemon` configuration key

### Replace migration-warning interfaces

Prefer current equivalents when touching commands that still warn:

| Older interface | Current form |
| --- | --- |
| `--parallel` | Express coexistence with task-level `persistent` and `with` |
| `--no-cache` | `--cache=local:r,remote:r` |
| `TURBO_REMOTE_ONLY` or `--remote-only` | `--cache=remote:rw` |
| `TURBO_REMOTE_CACHE_READ_ONLY` or `--remote-cache-read-only` | `--cache=local:rw,remote:r` |
| `turbo prune --scope web` | `turbo prune web` |
| `.png`, `.jpg`, or `.pdf` graph output | `.svg`, `.html`, `.mermaid`, or `.dot` |
| `.json` graph output | `turbo query` |
| `turbo-ignore` | `turbo query affected` |

Treat `turbo scan` as obsolete; it has no replacement.

### Reason about task cycles, not package cycles

Do not reject a repository merely because its Package Graph contains a cycle.
Validate the Task Graph. A task without cyclic task dependencies can run, while
a dependency such as `"dependsOn": ["^build"]` can still form an invalid cycle.

### Adopt 3.0 behavior selectively

Enable each `futureFlags` entry independently and expect it to change the global
hash. Read the configuration and execution references before enabling:

- `globalConfiguration`
- `affectedUsingTaskInputs`
- `watchUsingTaskInputs`
- `filterUsingTasks`
- `pruneIncludesGlobalFiles`
- `errorsOnlyShowHash`
- `longerSignatureKey`
- `experimentalObservability`

## Task orchestration quick reference

### Run persistent sidecars

Use `with` when long-running tasks must coexist. Do not use `dependsOn` for
services that should start alongside one another.

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

### Continue only through successful dependencies

Keep unrelated work running after a failure while blocking dependents of failed
tasks:

```bash
turbo run test --continue=dependencies-successful
```

### Let long-running tasks clean up

On `SIGINT` or `SIGTERM`, allow Turborepo to forward the signal and wait for task
cleanup handlers. Use a second `Ctrl+C` only when an immediate forced exit is
required.

## Configuration and hashing quick reference

### Extend arrays instead of replacing them

Use `$TURBO_EXTENDS$` to retain inherited values in a package configuration.
Without it, a local array replaces the inherited array.

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

### Hash generated inputs after dependencies

Use a structured input with `mode: "jit"` for files produced by an upstream
task. Use `mode: "dependencyOutputs"` when a downstream task should hash selected
outputs rather than upstream implementation details.

```jsonc
{
  "tasks": {
    "build": {
      "dependsOn": ["codegen"],
      "inputs": [
        "$TURBO_DEFAULT$",
        "!src/generated/**",
        { "mode": "jit", "globs": ["src/generated/**"] }
      ]
    },
    "check-types": {
      "dependsOn": ["^check-types"],
      "inputs": [{
        "mode": "dependencyOutputs",
        "globs": ["dist/**/*.d.ts"],
        "from": ["^check-types"]
      }]
    }
  }
}
```

## Scope and cache quick reference

### Intersect affected work with filters

Combine `--affected` and `--filter`; Turborepo selects only tasks or packages in
both scopes. Use a negated filter to remove a package from the affected set.

```bash
turbo run build --affected --filter=web
turbo run build --affected --filter=!docs
turbo query ls --affected --filter=my-app
```

### Bound the local cache

Opt into automatic eviction with top-level age and size limits:

```jsonc
{
  "cacheMaxAge": "7d",
  "cacheMaxSize": "10GB"
}
```

At the start of `turbo run`, eviction removes expired artifacts and then removes
the oldest remaining artifacts needed to satisfy the size limit in a background
thread.

## Query and diagnostics quick reference

Use `turbo query` for stable, machine-readable repository inspection:

```bash
turbo query
turbo query --schema
turbo query affected --tasks build
turbo query affected --packages
turbo query ls web --output=json
```

Run `turbo devtools` when live Package Graph and Task Graph views will make direct
and transitive relationships or cache misses easier to explain. For automation,
prefer query output, dry runs, and structured logs over scraping terminal output.
