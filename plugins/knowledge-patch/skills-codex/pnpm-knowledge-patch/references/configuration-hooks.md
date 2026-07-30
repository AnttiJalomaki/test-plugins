# Configuration, patches, and hooks

## Configuration dependencies

`pnpm.configDependencies` are installed before ordinary, development, and
optional dependencies. In early pnpm 10 they cannot have dependencies or
lifecycle scripts, and every entry must use an exact version plus an integrity
checksum (`2025-01`):

```json
{
  "pnpm": {
    "configDependencies": {
      "my-configs": "1.0.0+sha512-30iZtAPgz+LTIYoeivqYo853f02jBYSd5uGnGpkFV0M3xOt9aN73erkgYAmZU43x4VfqcnLxW9Kpg3R5LC4YYw=="
    }
  }
}
```

Install one without manually constructing the checksummed entry:

```sh
pnpm add --config my-config@1.0.0
```

In pnpm 11.1–11.3, config dependencies may have one level of
`optionalDependencies`, filtered by `os`, `cpu`, and `libc`. Those optional
packages must use exact versions, and the environment lockfile records every
platform variant (`11.1-11.3`).

## Workspace configuration

Starting in early pnpm 10, settings formerly nested under `pnpm` in
`package.json` may be top-level keys in `pnpm-workspace.yaml`; the file does not
need a `packages` field (`2025-02`):

```yaml
onlyBuiltDependencies:
  - esbuild
  - fuse-native
```

`pnpm-workspace.yaml` later accepts every `.npmrc` setting as a camelCase key,
including environment variables in setting names and values (`2025-03`).
`pnpm config get` and `pnpm config list` include these settings.
`pnpm config set --location=project` writes to the workspace file when there is
no project `.npmrc`.

```yaml
verifyDepsBeforeRun: install
optimisticRepeatInstall: true
publicHoistPattern:
  - "*types*"
  - "!@types/react"
```

When pnpm updates `pnpm-workspace.yaml`, current versions preserve comments,
string formatting, whitespace, and hand-maintained layout (`2026-03`).

pnpm 11 makes the separation authoritative: `.npmrc` is for authentication and
registries; all other settings live in `pnpm-workspace.yaml`. The root
`package.json#pnpm` field is ignored. See
[migration-pnpm-11.md](migration-pnpm-11.md).

## Structured config access

`pnpm config get` and `set` accept property paths, including leading-dot and
bracket notation. Object values print as INI strings by default; `--json`
serializes retrieved objects or parses a value being set (`2025-08`):

```sh
pnpm config get catalog.react
pnpm config get 'packageExtensions["@babel/parser"].peerDependencies["@babel/types"]'
pnpm config set .ignoreScripts true
pnpm config get --json catalog
```

Reporter output for `pnpm config` subcommands goes to stderr in pnpm 11.6–11.9,
so scripts can capture stdout without progress text (`11.6-11.9`).

## Patch selection

`patchedDependencies` accepts package-name, range, and exact-version keys
(`2025-03`). Exact versions take precedence over ranges, and ranges take
precedence over name-only entries. Do not overlap ranges.

```yaml
patchedDependencies:
  foo: patches/foo-1.patch
  foo@^2.0.0: patches/foo-2.patch
  foo@2.1.0: patches/foo-3.patch
```

`foo@*` matches like a name-only key, except that patch-application failures
are not ignored.

`pnpm.allowNonAppliedPatches` was renamed to `pnpm.allowUnusedPatches`; the old
name works in pnpm 10 but warns. `ignorePatchFailures` is tri-state in pnpm 10:

- Unset: exact-version and range failures are errors; name-only failures are
  ignored.
- `false`: every patch failure is an error.
- `true`: every patch failure is a warning.

pnpm 11 removes `ignorePatchFailures`; all failures throw. pnpm 11.4–11.5 also
rejects patch files whose `diff --git` headers write, delete, or rename paths
outside the patched package (`11.4-11.5`).

## pnpmfile hooks

The experimental `hooks.updateConfig` hook can rewrite pnpm settings
(`2025-04`):

```js
module.exports = {
  hooks: {
    updateConfig: (config) => ({ ...config, nodeLinker: "hoisted" }),
  },
};
```

Local pnpmfiles may also define `preResolution`, `importPackage`, and `fetchers`.
The `pnpmfile` setting accepts a list of hook files (`2025-07`):

```yaml
pnpmfile:
  - ./hooks/first.pnpmfile.cjs
  - ./hooks/second.pnpmfile.cjs
```

Config dependencies named `@pnpm/plugin-*` or `pnpm-plugin-*` load their
`pnpmfile.cjs` automatically in alphabetical order. In pnpm 10.15 and newer,
`@scope/pnpm-plugin-*` packages are also discovered (`2025-08`). List files
explicitly whenever hook order matters.

pnpm 11 supports `.pnpmfile.mjs`; when both module formats exist, the `.mjs`
file takes precedence and only one loads (`11.0.0`).

## Finder and packing hooks

Top-level `finders` define predicates for `pnpm list` and `pnpm why`, selected
with `--find-by=<name>` (`2025-09`). A finder returns `true`, or returns a string
that both matches and appears as extra result information.

```js
module.exports = {
  finders: {
    react17: (ctx) =>
      ctx.readManifest().peerDependencies?.react === "^17.0.0",
  },
};
```

```sh
pnpm why --find-by=react17
```

`hooks.beforePacking` runs immediately before `pnpm pack` or `pnpm publish`
creates a tarball. It returns the manifest to publish without changing the
local `package.json` (`2026-01-02`):

```js
module.exports = {
  hooks: {
    beforePacking(pkg) {
      delete pkg.devDependencies
      return pkg
    }
  }
}
```

## Delegating custom fetches

A custom fetcher may return `{ delegate: <resolution> }` to rewrite a
resolution and invoke pnpm's built-in fetcher (`11.10-11.17`):

```js
return { delegate: resolution }
```

This is the portable delegation form for pacquet, whose hook code cannot
receive `cafs` and `fetchers`.
