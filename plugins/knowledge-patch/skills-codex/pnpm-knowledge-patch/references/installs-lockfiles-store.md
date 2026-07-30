# Installs, lockfiles, linkers, and stores

## Install semantics

In pnpm 10, `NODE_ENV=production` no longer omits dependency types during
install; all dependency categories are installed (`2025-01`).

`publicHoistPattern` no longer implicitly includes names containing `eslint` or
`prettier`. Configure explicit public hoisting if those packages must be visible
at the root of `node_modules`.

`verifyDepsBeforeRun` controls what happens when `node_modules` is stale before
a script. Values are `install`, `warn`, `prompt`, `error`, or `false`:

```ini
verify-deps-before-run=install
```

Commands that should not change `node_modules`, including
`pnpm install --lockfile-only`, do not validate or purge it. This verification
is unsupported with `nodeLinker: pnp`; combining them warns (`2025-04`).

## Lockfile compatibility

pnpm 10 cannot directly convert a v6 lockfile to v9. Perform the conversion
with the pnpm 9 CLI before upgrading (`2025-01`).

pnpm 10.33 can read pnpm 11's two-document YAML lockfile. It ignores the first
document, which contains pnpm-version integrities and config-dependency
resolutions (`2026-03`).

In CI, an incompatible lockfile is fatal when frozen-lockfile mode is enabled;
non-frozen CI installs keep their fallback behavior.

```sh
pnpm install --frozen-lockfile
```

## Inspect and recover lockfiles

`pnpm list --lockfile-only` reads the expected graph from the lockfile instead
of `node_modules` (`2025-11`):

```sh
pnpm list --lockfile-only
```

When the root `pnpm-lock.yaml` is missing but
`node_modules/.pnpm/lock.yaml` still satisfies the manifest, pnpm 11.4–11.5
regenerates the root lockfile from that installed snapshot without resolving
again. A frozen install still refuses to run without the root lockfile
(`11.4-11.5`).

pnpm 11.10–11.17 may read `pnpm-lock.yaml` through a symlink. If install makes
no lockfile change, it leaves the symlink untouched; if a change is required,
pnpm refuses to write through it (`11.10-11.17`).

## Tarball integrity

pnpm records an integrity checksum on first fetch of an HTTP tarball dependency,
so later installs can detect different content served at the same URL
(`2025-12`).

Starting with pnpm 10.34, a checksum mismatch in a non-frozen install fails with
`ERR_PNPM_TARBALL_INTEGRITY` instead of re-resolving and overwriting the locked
checksum (`10.34.0`). After validating the registry change, opt in narrowly:

```sh
pnpm install --update-checksums
```

`--force`, `pnpm update`, and `--fix-lockfile` do not bypass the error.

In pnpm 11.4–11.5, a tarball lockfile entry without integrity fails while the
lockfile is read with `ERR_PNPM_MISSING_TARBALL_INTEGRITY`, including frozen
installs. Git-hosted and local `file:` tarballs are exempt.

When a registry generates a tarball on demand and omits checksums from metadata,
pnpm downloads it, computes integrity, and records the result (`11.6-11.9`).
For this reason, `--lockfile-only` may still download such a tarball.

## Git and alias validation

pnpm 11.4–11.5 requires the `commit` field in a Git resolution to be a full
40-character hexadecimal SHA. Dependency aliases containing path-traversal
segments are rejected while reading manifests and while linking into
`node_modules` (`11.4-11.5`).

Patch headers may not target paths outside the package being patched; see
[configuration-hooks.md](configuration-hooks.md).

## Global virtual store

`enableGlobalVirtualStore` links project `node_modules` entries to a shared
virtual store at `<store-path>/links` instead of `node_modules/.pnpm`
(`2025-05-06`). Packages are keyed by dependency-graph hashes, allowing sharing
between projects.

```yaml
enableGlobalVirtualStore: true
```

pnpm disables this mode automatically in CI. The separate `ci` setting can
explicitly control CI detection. It may also be enabled globally:

```sh
pnpm config -g set enable-global-virtual-store true
```

Global-virtual-store projects register under `{storeDir}/v10/projects/`, so
`pnpm store prune` can find packages reachable from project and workspace
`node_modules` and remove unused entries from `links/` (`2025-12`). Unscoped
packages now live under an `@` directory; tooling must not assume the older
layout.

`pnpm deploy` ignores `enableGlobalVirtualStore` and builds a virtual store in
the deployment directory so output is self-contained (`2026-01-02`).

## Store v11 and read-only operation

pnpm 11 uses one SQLite package index at `$STORE/index.db`. Packages missing
from that index may be fetched again even if older per-package JSON indexes
exist (`11.0.0`).

`frozenStore`/`--frozen-store` suppresses every store write and installs from a
fully populated store on a read-only filesystem (`11.6-11.9`):

```sh
pnpm install --frozen-store --offline --frozen-lockfile
```

Required build outputs must already be present. Frozen-store mode is
incompatible with `--force` and pnpr servers. Its minimum runtime is Node.js
22.15 on the 22.x line, 23.11 on the 23.x line, or 24.

Reporter output from `pnpm store` subcommands goes to stderr, leaving stdout
safe for scripts (`11.6-11.9`).

## Virtual-store-only mode

`virtualStoreOnly` populates the virtual store without importer symlinks,
hoisting, binary linking, or lifecycle scripts (`11.0.0`). `pnpm fetch` uses
this mode internally.

```yaml
virtualStoreOnly: true
```

`pnpm fetch` skips local-directory dependencies using the `file:` protocol
(`2026-01-02`). This allows a Docker prefetch layer to run before copying local
directories, but the directories must exist for the later install.

## Hoisting and peer graph layout

When pnpm auto-installs a missing peer, pnpm 10.15 and newer prefer a matching
version already in the root workspace package's direct dependencies
(`2025-08`). This may change the selected peer version after an upgrade.

Enable `dedupePeers` to use version-only peer identifiers rather than full
dependency paths in peer suffixes. This avoids nested suffix chains and reduces
duplicates in recursive peer graphs (`2026-03`):

```yaml
dedupePeers: true
```

For `nodeLinker: hoisted`, `hoistingLimits` accepts (`11.4-11.5`):

- `none` (default): hoist as far as possible.
- `workspaces`: stop at each workspace package.
- `dependencies`: stop at each workspace package's direct dependencies.

```yaml
nodeLinker: hoisted
hoistingLimits: workspaces
```

## Node.js package maps

Isolated and hoisted installs create `node_modules/.package-map.json`
(`11.6-11.9`). Enable `nodeExperimentalPackageMap` to inject it into
pnpm-managed Node.js scripts. `nodePackageMapType: standard` exposes only
declared dependencies; `loose` also includes other reachable packages.

```yaml
nodeExperimentalPackageMap: true
nodePackageMapType: standard
```

## Pacquet backend

Installing `@pnpm/pacquet` as a config dependency delegates install
materialization while pnpm keeps resolution (`11.1-11.3`):

```sh
pnpm add @pnpm/pacquet --config
```

As of pnpm 11.2.2, `install`/`i` flags are forwarded to pacquet, but flags from
`add`, `update`, and `dedupe` are not.

With pacquet 0.11.7 or newer and the isolated linker, a plain non-frozen install
delegates both resolution and materialization (`11.6-11.9`). `add`, `update`,
and `remove` still resolve through pnpm; older pacquet versions retain the
resolve-then-materialize split.

## Previewing and tuning installs

`pnpm install --dry-run` performs full resolution, reports changes, writes
nothing to manifests, the lockfile, or `node_modules`, and exits 0 on a
successful preview (`11.6-11.9`).

```sh
pnpm install --dry-run
```

Use per-command `--cpu`, `--libc`, and `--os` on `install`, `add`, or `dlx` to
override `supportedArchitectures` (`2025-07`).

Set metadata and tarball warning thresholds with (`2025-10`):

```yaml
fetchWarnTimeoutMs: 10000
fetchMinSpeedKiBps: 50
```

The first is elapsed time; the second is minimum transfer speed in KiB/s.
