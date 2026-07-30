# Installation, lockfiles, and stores

This reference covers dependency installation, freshness checks, integrity,
lockfile interoperability, virtual stores, alternate materializers, and
platform selection. Relevant extraction markers include `2025-01`, `2025-04`,
`2025-05-06`, `2025-07`, `2025-10`, `2025-11`, `2025-12`, `2026-01-02`,
`2026-03`, `10.34.0`, `11.0.0`, `11.1-11.3`, `11.4-11.5`, and
`11.6-11.9`.

## Production and dependency freshness

In pnpm 10, `NODE_ENV=production` no longer omits dependency categories;
install still includes all dependencies. Select production-only behavior with
explicit pnpm options rather than relying on that environment variable.

`verifyDepsBeforeRun` controls what happens when `node_modules` is stale before
a script. Values are `install`, `warn`, `prompt`, `error`, and `false`.

```yaml
verifyDepsBeforeRun: install
```

Commands that should not alter `node_modules`, such as
`pnpm install --lockfile-only`, no longer validate or purge it. Freshness
verification is unsupported with `nodeLinker: pnp`; the combination warns, so
PnP projects need another validation strategy.

## Lockfile migrations and compatibility

pnpm 10 cannot directly convert lockfile v6 to v9. Use pnpm 9 for that
conversion before moving the project to pnpm 10.

pnpm 10.33 can consume pnpm 11's two-document `pnpm-lock.yaml`: it ignores the
first document used for pnpm-version integrities and config-dependency
resolutions.

Frozen CI installs now fail when the lockfile format is incompatible:

```sh
pnpm install --frozen-lockfile
```

Non-frozen CI retains fallback behavior. If the root lockfile is missing but
`node_modules/.pnpm/lock.yaml` still matches the manifest, pnpm 11 can rebuild
the root lockfile from that installed snapshot without a new resolution. A
frozen install still refuses a missing root lockfile.

pnpm can read a root lockfile through a symlink. If install makes no lockfile
changes, it does not rewrite the link, which supports frozen staged build
sandboxes. If a change is required, pnpm refuses to write through the symlink.

## Integrity failures

From pnpm 10.34, a tarball integrity mismatch is fatal even in a non-frozen
install. Review the source, then narrowly accept current registry bytes with:

```sh
pnpm install --update-checksums
```

`--force`, `pnpm update`, and `--fix-lockfile` do not bypass the mismatch or
rewrite the checksum.

pnpm 11 also fails immediately with `ERR_PNPM_MISSING_TARBALL_INTEGRITY` when a
lockfile tarball entry has no integrity value, including frozen installs. Git
tarballs and local `file:` tarballs are exempt.

When a registry generates a tarball on demand and omits metadata checksums,
pnpm downloads it, computes integrity, and records that value. Therefore even
`--lockfile-only` may download such a tarball to make the lockfile verifiable.
HTTP tarballs likewise gain an integrity hash on first fetch so later installs
detect changed bytes at the same URL.

Git resolution `commit` fields must be full 40-character hexadecimal SHAs.
Short SHAs and symbolic names are rejected before Git runs.

## Dry-run and virtual-store-only installs

Preview full resolution without changing manifests, lockfiles, or
`node_modules`:

```sh
pnpm install --dry-run
```

A successful preview exits zero. For a real store population that avoids
importer symlinks, hoisting, binary links, and lifecycle scripts, use:

```yaml
virtualStoreOnly: true
```

`pnpm fetch` uses virtual-store-only behavior internally. It skips local
directory dependencies expressed with `file:`, which allows a container
prefetch layer to run before local source directories are copied. Those
directories must exist for the later install.

## Experimental global virtual store

With `enableGlobalVirtualStore`, project `node_modules` entries link into
`<store-path>/links` rather than `node_modules/.pnpm`. Entries are keyed by
dependency-graph hashes so projects can share them.

```yaml
enableGlobalVirtualStore: true
```

It may also be enabled globally:

```sh
pnpm config -g set enable-global-virtual-store true
```

pnpm automatically disables the mode in CI; the independent `ci` setting can
override environment detection. `pnpm deploy` ignores the global mode and
creates a deployment-local virtual store.

Global-store projects register under `{storeDir}/v10/projects/`. This lets
`pnpm store prune` mark packages reachable from project and workspace
`node_modules` and remove unused entries below `links/`. Unscoped packages now
live under an `@` directory; tools that assumed the older layout must adapt.

Reporter output from `pnpm store` commands goes to stderr in newer pnpm 11 so
stdout remains script-friendly.

## Store v11

pnpm 11 stores its package index in one SQLite database at `$STORE/index.db`.
Packages present in the content-addressed store but absent from the new index
are refetched on demand rather than rediscovered from older per-package JSON
indexes.

## Frozen read-only stores

`frozenStore` or `--frozen-store` installs from a prepopulated store without
writing to it. Combine it with offline and frozen-lockfile operation:

```sh
pnpm install --frozen-store --offline --frozen-lockfile
```

Required build outputs must already be present. Frozen-store mode is
incompatible with `--force` and pnpm servers. It requires Node.js 22.15+ on the
22 line, 23.11+ on the 23 line, or 24+.

## Pacquet install backend

Install `@pnpm/pacquet` as a config dependency to delegate materialization
while pnpm performs resolution:

```sh
pnpm add @pnpm/pacquet --config
```

As of pacquet support in pnpm 11.2.2, `install`/`i` flags are forwarded, but
flags supplied to `add`, `update`, and `dedupe` are not. With pacquet 0.11.7+
and the isolated linker, a normal non-frozen install delegates both resolution
and materialization. `add`, `update`, and `remove` still resolve in pnpm; older
pacquet versions retain pnpm resolution followed by pacquet materialization.

## Architecture selection

Override configured supported architectures for one `install`, `add`, or
`dlx` command with `--cpu`, `--libc`, and `--os`. This is useful for preparing
dependencies for a target image without permanently changing
`supportedArchitectures`.

## Slow-network diagnostics

Set metadata latency and tarball throughput warning thresholds:

```yaml
fetchWarnTimeoutMs: 10000
fetchMinSpeedKiBps: 50
```

These warn about slow network behavior without changing resolution semantics.

## Prefix and workspace discovery

When invoking pnpm from outside a project, `--prefix=<dir>` participates in
workspace-root detection and loads that directory's `pnpm-workspace.yaml`:

```sh
pnpm --prefix=./project install
```

## Inspect without installing

`pnpm list --lockfile-only` reads expected package information from the
lockfile rather than `node_modules`:

```sh
pnpm list --lockfile-only
```

Use it when the desired question is about the resolved installation rather
than the current filesystem state.
