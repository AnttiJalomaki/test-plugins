# Migration and configuration

Use this reference when updating configuration syntax, inheriting package
configuration, enabling future behavior, or preparing automation for the next
major release.

## Version-matched schemas and JSON syntax

The installed `turbo` package contains `schema.json` (since 2.4.0). Point the
configuration at that file when editor completion and validation should match
the installed CLI:

```json
{
  "$schema": "./node_modules/turbo/schema.json"
}
```

For a package-level `turbo.json`, adjust the relative path, for example
`../../node_modules/turbo/schema.json`.

The `turbo.jsonc` filename accepts comments (since 2.5.0):

```jsonc
{
  "tasks": {
    "test": {
      // Build dependencies before testing.
      "dependsOn": ["^build"]
    }
  }
}
```

Ordinary `turbo.json` also accepts trailing commas (since 2.6.0), so comments
are the main reason to choose the `.jsonc` filename.

## Package configuration inheritance

A package-level `turbo.json` may extend another workspace package by naming it
in `extends` (since 2.7.0). `//` continues to mean the root configuration:

```json
{
  "extends": ["//", "web"]
}
```

An array in local configuration normally replaces the inherited array. Insert
`$TURBO_EXTENDS$` to retain the inherited values and append local entries:

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

Task definitions accept a human-readable `description` (since 2.8.0).
Descriptions provide context to people and tools but do not affect execution:

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

## Environment-variable linting

`eslint-config-turbo` and `eslint-plugin-turbo` support ESLint Flat Config
while retaining ESLint 8 compatibility (since 2.4.0). Spread the shareable
configuration into the exported array:

```js
export default [
  ...turboConfig,
];
```

For direct plugin use, register `turbo` in the flat-config `plugins` object:

```js
export default [
  {
    plugins: { turbo },
    rules: {
      "turbo/no-undeclared-env-vars": "error",
    },
  },
];
```

Biome 2.3.10 and newer detects a Turborepo project from repository
dependencies (since Turborepo 2.7.0). Its `noUndeclaredEnvVars` rule remains in
the nursery group and must be enabled explicitly:

```json
{
  "linter": {
    "rules": {
      "nursery": {
        "noUndeclaredEnvVars": "error"
      }
    }
  }
}
```

Both integrations catch environment-variable reads that can otherwise cause an
incorrect cache hit.

## Future flags

Opt-in behaviors live under `futureFlags` (since 2.9.0). Flags are independent,
but enabling any one changes the global hash.

- `globalConfiguration` moves formerly top-level global settings under
  `global`.
- `affectedUsingTaskInputs` selects affected tasks from their `inputs` rather
  than selecting every task in a changed package.
- `watchUsingTaskInputs` reruns only tasks whose `inputs` match a changed file.
- `filterUsingTasks` resolves filters at task level, using task inputs for Git
  ranges and the Task Graph for `...` traversal.
- `pruneIncludesGlobalFiles` copies files matched by `globalDependencies` into
  prune output.
- `errorsOnlyShowHash` includes task hashes when `outputLogs` is
  `"errors-only"`.
- `longerSignatureKey` requires `TURBO_REMOTE_CACHE_SIGNATURE_KEY` to contain
  at least 32 bytes.
- `experimentalObservability` gates the OpenTelemetry configuration.

Enable only the behavior being tested, and account for the resulting global
hash change when comparing cache results.

## Deprecations for the next major release

The following interfaces still work with warnings in 2.9.0 but should be
migrated:

- Remove `turbo scan`; it is obsolete and has no replacement.
- Replace `--parallel` with task-level `persistent` and `with`.
- Replace `--no-cache` with `--cache=local:r,remote:r`.
- Replace `TURBO_REMOTE_ONLY` and `--remote-only` with
  `--cache=remote:rw`.
- Replace `TURBO_REMOTE_CACHE_READ_ONLY` and `--remote-cache-read-only` with
  `--cache=local:rw,remote:r`.
- Replace `.png`, `.jpg`, and `.pdf` graph output with `.svg`, `.html`,
  `.mermaid`, or `.dot`.
- Replace JSON graph output with `turbo query`.
- Replace `turbo prune --scope web` with `turbo prune web`.

`turbo-ignore` is deprecated in favor of `turbo query affected` (since 2.9.0).

`turbo run` and `turbo watch` no longer use the daemon (since 2.9.0).
`TURBO_DAEMON`, `--daemon`, `--no-daemon`, and the `daemon` configuration key
are deprecated because they no longer control execution.

## Migration codemod

Run the migration codemod to upgrade a repository:

```bash
npx @turbo/codemod migrate
```

The codemod understands package-manager catalogs (since 2.10.0), so catalog
entries participate in migration instead of requiring a separate manual
rewrite.
