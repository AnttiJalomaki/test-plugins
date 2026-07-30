# Task execution and inputs

Use this reference when defining long-running tasks, continuing after failures,
diagnosing cycles, controlling watch caching, or hashing files that appear
after dependencies run.

## Persistent tasks and sidecars

A persistent task can start other long-running tasks with `with` (since
2.5.0):

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

`with` expresses runtime coexistence. It does not impose the completion
ordering of `dependsOn`. Dry-run and summary output include these sidecar
relationships (since 2.6.0), making it possible to inspect the resulting
execution plan.

## Dependency-safe continuation

Use `dependencies-successful` to continue independent work after a failure
without running a task whose dependencies failed (since 2.5.0):

```bash
turbo run test --continue=dependencies-successful
```

This is stricter than merely continuing past an error: every dependency of a
candidate task must have succeeded.

## Package cycles and Task Graph cycles

Circular-package diagnostics list sets of dependency edges where removing any
one complete set will break the cycle (since 2.4.0). Use the suggested edge
sets to choose a coherent refactor rather than deleting an arbitrary single
edge.

A Package Graph cycle no longer causes an automatic exit (since 2.9.0).
Turborepo validates the Task Graph instead:

```json
{
  "tasks": {
    "simple-task": {},
    "build": { "dependsOn": ["^build"] }
  }
}
```

`simple-task` can run even when packages form a cycle because it creates no
cyclic task relationship. `^build` still errors when it turns that package
cycle into a Task Graph cycle.

## Root-anchored inputs

Use `$TURBO_ROOT$` to anchor a glob at the workspace root (since 2.5.0):

```json
{
  "tasks": {
    "build": {
      "inputs": ["$TURBO_ROOT$/important-file.txt"]
    }
  }
}
```

This avoids package-depth-dependent paths such as `../important-file.txt`.

## Deferred task-input hashing

Structured input objects can defer part of a task hash until upstream work has
completed (since 2.10.0).

Use `mode: "jit"` for files produced by a dependency:

```jsonc
{
  "tasks": {
    "codegen": {
      "cache": false
    },
    "build": {
      "dependsOn": ["codegen"],
      "inputs": [
        "$TURBO_DEFAULT$",
        "!src/generated/**",
        {
          "mode": "jit",
          "globs": ["src/generated/**"]
        }
      ]
    }
  }
}
```

The ordinary input set excludes the generated directory, then the JIT entry
hashes it after `codegen` completes. Keep the dependency edge: hashing mode
does not establish execution ordering.

Use `mode: "dependencyOutputs"` when downstream work depends on selected
outputs of upstream tasks:

```jsonc
{
  "tasks": {
    "check-types": {
      "dependsOn": ["^check-types"],
      "outputs": ["dist/**"],
      "inputs": [
        "$TURBO_DEFAULT$",
        {
          "mode": "dependencyOutputs",
          "globs": ["dist/**/*.d.ts"],
          "from": ["^check-types"]
        }
      ]
    }
  }
}
```

This allows a downstream task to remain cached when an upstream
implementation changes but the selected public outputs, such as declaration
files, remain identical.

## Watch Mode cache writes

Watch Mode can write task results to cache when explicitly enabled (since
2.4.0). A watched task argument is required:

```bash
turbo watch dev --experimental-write-cache
```

Without the flag, do not assume watch results populate the cache.

The `watchUsingTaskInputs` future flag can make watch rerun only tasks whose
declared inputs match changed files (since 2.9.0). Enabling the flag also
changes the global hash.

## Graceful shutdown

On `SIGINT` or `SIGTERM`, Turborepo forwards the signal to running tasks and
waits for their cleanup handlers to complete (since 2.10.0). No configuration
is required.

Press `Ctrl+C` a second time only when immediate termination is necessary; the
second signal bypasses the graceful wait.
