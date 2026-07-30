# Workspaces, catalogs, linking, and deploy

## Link override behavior

Early pnpm 10 `pnpm link` writes an override to the root `package.json`; in a
workspace this makes the link apply to every project (`2025-01`). To create a
global link in that era, run `pnpm link` inside the package directory rather
than `pnpm link -g`.

Later pnpm 10 writes the root override to `pnpm-workspace.yaml`, not
`package.json` (`2025-04`). `pnpm audit --fix` likewise updates overrides in the
workspace file.

pnpm 11 requires a relative or absolute path for `pnpm link`; package-name
lookup from the global store is removed. Global link commands are also removed;
see [migration-pnpm-11.md](migration-pnpm-11.md).

## Injected workspace dependencies

`injectWorkspacePackages: true` hard-links all local workspace dependencies
instead of symlinking them. pnpm 10 requires this setting for `pnpm deploy`
(`2025-01`).

```yaml
injectWorkspacePackages: true
```

`syncInjectedDepsAfterScripts` names root-configured scripts after which
`pnpm run` synchronizes an injected package's files into consumers
(`2025-02`):

```ini
sync-injected-deps-after-scripts[]=compile
```

## Deployment lockfiles and stores

pnpm 10 deploy attempts to derive a dedicated deployment lockfile from the
shared workspace lockfile. If it cannot, it falls back to no deployment
lockfile; `forceLegacyDeploy: true` forces that fallback (`2025-01`).

Deployment always creates its virtual store inside the output directory,
ignoring `enableGlobalVirtualStore`, so the deployed tree is self-contained
(`2026-01-02`). Current pnpm also supports workspaces whose dependencies use
catalogs (`11.10-11.17`).

## Catalog-aware additions

`pnpm add` writes a `catalog:` specifier when a requested dependency and range
match the default workspace catalog. Omitting the range also selects the
catalog entry. A nonmatching request remains a direct specifier (`2025-01`).

Save entries explicitly (`2025-05-06`):

```sh
pnpm add --save-catalog lodash
pnpm add --save-catalog-name=testing vitest
```

The manifest receives `catalog:` for the default catalog or
`catalog:<name>` for a named catalog.

## Updating catalogs and controlling additions

`pnpm update` updates `catalog:` dependencies and writes the new ranges to
`pnpm-workspace.yaml` (`2025-05-06`). `catalogMode` controls additions:

- `strict`: reject a request outside the catalog range.
- `prefer`: use a compatible catalog entry, otherwise save a direct dependency.
- `manual` (default): do not add dependencies to catalogs automatically.

```yaml
catalogMode: strict
```

Set `cleanupUnusedCatalogs: true` to remove catalog entries no longer used
whenever install runs (`2025-08`).

## Catalogs outside normal installs

`pnpm dlx` and `pnpx` accept `catalog:` so ephemeral commands can use workspace
catalog versions (`2026-01-02`):

```sh
pnpm dlx shx@catalog:
```

Current `pnpm deploy` supports catalog-managed dependencies. `pnpm update
--changeset` also includes catalog consumers in generated release intents; see
[release-management.md](release-management.md).

## Workspace protocol and peer ranges

A dependency may use bare `workspace:`; pnpm treats it as `workspace:*` and
replaces it with a concrete version when publishing (`2026-01-02`):

```json
{
  "dependencies": {
    "foo": "workspace:"
  }
}
```

Starting in pnpm 10.1, `workspace:` and `catalog:` specifiers can participate in
wider `peerDependencies` ranges (`2025-01`).

Current peer dependencies may also use named-registry, `npm:` alias, `file:`,
Git, or URL schemes (`11.10-11.17`). Matching uses the embedded range—for
example `5.x.x` in `work:5.x.x` or `^5` in `npm:bar@^5`—and `*` when no version
is present. Bare `name@version` peer values remain invalid.

## Recursive workspace execution

`pnpm -r pack` packs every workspace project (`2025-05-06`):

```sh
pnpm -r pack
```

The default `workspaceConcurrency` is
`Math.min(os.availableParallelism(), 4)`, so recursive execution uses at most
four tasks unless configured otherwise.

`pnpm run` accepts a slash-delimited regular expression to run all matching
scripts. `--sequential`/`-s` forces `workspaceConcurrency` to one across and
within packages (`11.10-11.17`):

```sh
pnpm run --sequential "/^build:.*/"
```

## Prefix-based workspace discovery

Starting with pnpm 10.34, `--prefix=<dir>` participates in workspace-root
detection even when pnpm starts outside the project, and loads that directory's
`pnpm-workspace.yaml` (`10.34.0`):

```sh
pnpm --prefix=./project install
```
