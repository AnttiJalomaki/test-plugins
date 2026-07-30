# Configuration and Task Inputs

## Local schema and syntax

Point root configuration at the schema shipped in the installed package so
completion and validation match the repository's Turborepo version (since
2.4.0):

```json
{
  "$schema": "./node_modules/turbo/schema.json"
}
```

Adjust the relative path in package-level files, for example
`../../node_modules/turbo/schema.json`.

Use `turbo.jsonc` for JSON-with-comments syntax (since 2.5.0). A regular
`turbo.json` also accepts trailing commas (since 2.6.0), so a separate JSONC
filename is not necessary solely for trailing commas.

## Environment and cache precedence

Expect `DISPLAY` to pass through by default (since 2.4.0). Use negation in
`passThroughEnv` to exclude built-in variables and variables inherited from
`globalPassThroughEnv`. Treat force mode as authoritative: it overrides other
cache settings, including `remoteCache.enable`, rather than being overridden by
them.

## Workspace-root input globs

Anchor an input at the workspace root with `$TURBO_ROOT$` instead of brittle
package-depth-relative paths (since 2.5.0):

```json
{
  "tasks": {
    "build": {
      "inputs": ["$TURBO_ROOT$/important-file.txt"]
    }
  }
}
```

## Package configuration inheritance

Allow a package-level `turbo.json` to extend the root (`//`) and another
workspace package by name (since 2.7.0):

```json
{
  "extends": ["//", "web"]
}
```

Local arrays replace inherited arrays unless they contain `$TURBO_EXTENDS$`.
Use the marker to append local values:

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

## Task descriptions

Add `description` to a task for human and tool context (since 2.8.0). The value
is informational and does not change task execution:

```json
{
  "tasks": {
    "build": {
      "description": "Compiles TypeScript and bundles the application",
      "outputs": ["dist/**"]
    }
  }
}
```

## Future Flags

Enable opt-in behaviors independently under `futureFlags` (since 2.9.0). Each
enabled flag affects the global hash.

- `globalConfiguration` moves formerly top-level global settings under
  `global`.
- `affectedUsingTaskInputs` selects individual affected tasks using their
  `inputs`, instead of selecting every task in a changed package.
- `watchUsingTaskInputs` reruns only tasks whose `inputs` match changed files.
- `filterUsingTasks` resolves filters at task level, using `inputs` for Git
  ranges and the Task Graph for `...` traversal.
- `pruneIncludesGlobalFiles` copies files matched by `globalDependencies` into
  prune output.
- `errorsOnlyShowHash` includes task hashes when `outputLogs` is
  `"errors-only"`.
- `longerSignatureKey` requires `TURBO_REMOTE_CACHE_SIGNATURE_KEY` to contain at
  least 32 bytes.
- `experimentalObservability` gates OpenTelemetry configuration.

## Deferred task-input hashing

Use structured input objects for generated or dependency-produced files (since
2.10.0). A `jit` input is hashed after task dependencies finish:

```jsonc
{
  "tasks": {
    "codegen": { "cache": false },
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

Use `dependencyOutputs` to hash selected outputs from task dependencies. This
keeps downstream tasks cached when an upstream implementation changes but its
selected outputs do not:

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
