# Configuration and workspaces

This reference covers workspace policy, configuration hooks, catalogs, local
packages, deploy, hoisting, and peers. Relevant extraction markers include
`2025-01`, `2025-02`, `2025-03`, `2025-04`, `2025-05-06`, `2025-07`,
`2025-08`, `2026-01-02`, `2026-03`, `11.0.0`, `11.4-11.5`, and
`11.10-11.17`.

## Workspace configuration

In later pnpm 10, `pnpm-workspace.yaml` accepts every `.npmrc` setting using
camelCase keys. It may contain settings without a `packages` field, so it can
exist solely for workspace policy such as build approvals.

```yaml
verifyDepsBeforeRun: install
optimisticRepeatInstall: true
publicHoistPattern:
  - "*types*"
  - "!@types/react"
onlyBuiltDependencies:
  - esbuild
```

Environment variables may appear in setting names and values.
`pnpm config get` and `pnpm config list` include workspace settings. When the
project has no `.npmrc`, this writes to `pnpm-workspace.yaml`:

```sh
pnpm config set --location=project verifyDepsBeforeRun install
```

When pnpm updates the workspace file, current versions preserve comments,
quoted-string style, whitespace, and other hand-maintained formatting.

In pnpm 11, the workspace file is authoritative for all settings except
registry and authentication data. Do not retain policy in the ignored `pnpm`
field of `package.json` or in `.npmrc`.

## Structured config access

`pnpm config get` and `set` accept dotted, leading-dot, and bracket paths.
Object values print as INI by default; `--json` serializes reads or parses a
written JSON value.

```sh
pnpm config get catalog.react
pnpm config get 'packageExtensions["@babel/parser"].peerDependencies["@babel/types"]'
pnpm config set .ignoreScripts true
pnpm config get --json catalog
```

Reporter output for `pnpm config` goes to stderr in newer pnpm 11, leaving
stdout suitable for scripts.

## pnpmfile hooks and config dependencies

The experimental `.pnpmfile.cjs` `hooks.updateConfig` hook may rewrite the
resolved pnpm settings. Local pnpmfiles may also define `preResolution`,
`importPackage`, and `fetchers` hooks.

```js
module.exports = {
  hooks: {
    updateConfig: (config) => ({ ...config, nodeLinker: "hoisted" }),
  },
}
```

The `pnpmfile` setting accepts multiple hook files:

```yaml
pnpmfile:
  - ./hooks/first.pnpmfile.cjs
  - ./hooks/second.pnpmfile.cjs
```

Config dependencies named `@pnpm/plugin-*`, `pnpm-plugin-*`, or later
`@scope/pnpm-plugin-*` have their `pnpmfile.cjs` loaded automatically in
alphabetical order. List files explicitly when exact ordering matters.

pnpm 11 also supports `.pnpmfile.mjs`. When `.mjs` and `.cjs` both exist,
`.mjs` wins and only one file is loaded. A custom fetcher may return
`{ delegate: resolution }` to rewrite a resolution and invoke pnpm's built-in
fetcher; this is also the portable delegation form when pacquet hook code does
not receive `cafs` and `fetchers`.

## Config dependencies

`pnpm.configDependencies` are installed before normal, development, and
optional project dependencies. Each root entry uses an exact version plus an
integrity checksum, and cannot itself have normal dependencies or lifecycle
scripts.

```json
{
  "pnpm": {
    "configDependencies": {
      "my-configs": "1.0.0+sha512-30iZtAPgz+LTIYoeivqYo853f02jBYSd5uGnGpkFV0M3xOt9aN73erkgYAmZU43x4VfqcnLxW9Kpg3R5LC4YYw=="
    }
  }
}
```

Prefer the CLI to generate that entry:

```sh
pnpm add --config my-config@1.0.0
```

pnpm 11 permits one level of `optionalDependencies` for config dependencies,
filtered by `os`, `cpu`, and `libc`. Those optional versions must be exact, and
the environment lockfile records all platform variants.

## Root links and overrides

Early pnpm 10 `pnpm link` wrote a root override into `package.json`, making the
link affect every workspace project. Later pnpm 10 writes that override to
`pnpm-workspace.yaml`; `pnpm audit --fix` likewise updates overrides stored
there. Inspect the pnpm version before assuming which file changes.

To create a global link in early pnpm 10, run `pnpm link` from the package
directory rather than `pnpm link -g`. pnpm 11 removes argument-free link and
`pnpm link --global`; use `pnpm add -g .` for the global-package case. Local
linking needs a filesystem path:

```sh
pnpm link ./foo
```

## Injected workspace dependencies and deploy

`injectWorkspacePackages: true` hard-links local workspace dependencies rather
than symlinking them. pnpm 10 requires injection for `pnpm deploy`. It attempts
to derive a dedicated lockfile from the shared workspace lockfile and falls
back to no deployment lockfile when derivation is impossible or
`forceLegacyDeploy: true` is selected.

For injected packages, name scripts after which consumers should be
resynchronized:

```ini
sync-injected-deps-after-scripts[]=compile
```

Set each synchronized script at the workspace root. Deployment ignores
`enableGlobalVirtualStore` and creates its virtual store inside the destination
so output is self-contained. Current pnpm 11 can deploy workspaces whose
dependencies use catalogs.

## Catalog behavior

When `pnpm add` finds the requested package and compatible range in the default
workspace catalog, it writes `catalog:`. Omitting the range also selects the
catalog entry; an incompatible request keeps a normal dependency specifier.

`catalogMode` controls automatic additions:

- `manual` (default): never add to a catalog automatically;
- `strict`: reject a request outside the catalog range;
- `prefer`: use a compatible catalog entry, otherwise save a direct specifier.

```yaml
catalogMode: strict
cleanupUnusedCatalogs: true
```

`pnpm update` updates `catalog:` dependencies and their declarations in
`pnpm-workspace.yaml`. `cleanupUnusedCatalogs` removes unreferenced catalog
entries during install.

Save directly to the default or named catalog:

```sh
pnpm add --save-catalog lodash
pnpm add --save-catalog-name=testing vitest
```

The consuming manifest receives `catalog:` or `catalog:<name>`. `pnpm dlx`
and `pnpx` also accept a workspace catalog specifier:

```sh
pnpm dlx shx@catalog:
```

## Workspace protocol and JSR

`workspace:` with no range is equivalent to `workspace:*`; publishing replaces
it with the concrete workspace package version.

The `jsr:` protocol installs JSR packages with an optional range. A scoped JSR
package is saved under its ordinary package name and converted to an npm alias
for publishing. `@jsr` defaults to `https://npm.jsr.io/` unless overridden by
`@jsr:registry`.

```sh
pnpm add jsr:@foo/bar
pnpm add jsr:@foo/bar@^0.1
```

```json
{
  "dependencies": {
    "@foo/bar": "jsr:^0.1.2"
  }
}
```

## Peer dependencies

pnpm 10.1 allows `workspace:` and `catalog:` specifiers to participate in wider
`peerDependencies` ranges. When auto-installing a missing peer, pnpm 10.15+
prefers a matching version already present in the root workspace package's
direct dependencies.

Enable `dedupePeers` to use version-only peer identifiers instead of full
dependency paths in peer suffixes. This avoids nested suffix chains and reduces
duplicates in recursive peer graphs:

```yaml
dedupePeers: true
```

Current peer dependency entries may use named-registry, `npm:` alias, `file:`,
Git, or URL schemes. Matching extracts the range inside the scheme—for example,
`5.x.x` from `work:5.x.x` or `^5` from `npm:bar@^5`—and uses `*` if no version
appears. Bare `name@version` remains invalid.

## Convergence overrides

An empty-range override selector changes only dependency edges whose declared
range accepts the exact override value. It converges compatible consumers
without forcing incompatible ones:

```yaml
overrides:
  "form-data@": 4.0.6
```

The replacement must be exact. pnpm warns when every declared range would
allow a newer convergence target.

## Hoisting and package maps

pnpm 10 no longer implicitly public-hoists package names containing `eslint`
or `prettier`. Configure `publicHoistPattern` when those dependencies need to
appear at the root of `node_modules`.

With `nodeLinker: hoisted`, set `hoistingLimits` to:

- `none` (default), hoist as far as possible;
- `workspaces`, stop at each workspace package;
- `dependencies`, stop at each workspace package's direct dependencies.

```yaml
nodeLinker: hoisted
hoistingLimits: workspaces
```

Both isolated and hoisted installs create `node_modules/.package-map.json`.
`nodeExperimentalPackageMap: true` injects it into pnpm-managed Node.js
processes. `nodePackageMapType: standard` exposes only declared dependencies;
`loose` includes other reachable packages.
