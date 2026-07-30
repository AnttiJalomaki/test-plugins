# Runtimes, scripts, initialization, and CLI behavior

## Package-manager version selection

`managePackageManagerVersions` is enabled by default in pnpm 10, so pnpm
selects its own version from the root `packageManager` field (`2025-01`):

```json
{
  "packageManager": "pnpm@10.1.0"
}
```

The version cannot have a `v` prefix; use `pnpm@10.4.0`, not
`pnpm@v10.4.0` (`2025-02`). When pnpm switches CLI versions, it disables
`managePackageManagerVersions` to avoid a second automatic switch
(`2025-10`).

Global installation of `pnpm` or `@pnpm/exe` with `pnpm add --global` fails in
pnpm 10; use `pnpm self-update`.

pnpm 11 replaces version-management strictness settings with `pmOnFail` and
can declare the package manager in `devEngines.packageManager`. The latter
accepts a range, resolves and stores a version in `pnpm-lock.yaml`, and reuses
it while compatible (`11.0.0`). `pnpm with` runs a one-off version while
bypassing project pins.

pnpm 11.10–11.17 can self-update or switch directly to the native Rust pnpm 12
build from `next-12`. The resulting install uses the unscoped `pnpm` package,
including upgrades from `@pnpm/exe` (`11.10-11.17`):

```sh
pnpm self-update next-12
```

## Project runtimes

`devEngines.runtime` provisions Node.js, Deno, or Bun for each workspace
project (`2025-07`). `pnpm install` resolves the requested range, records the
exact version and checksum in the lockfile, and uses the local runtime for
scripts. Initially `onFail` supports only `download`.

```json
{
  "devEngines": {
    "runtime": {
      "name": "node",
      "version": "^24.4.0",
      "onFail": "download"
    }
  }
}
```

The older `nodeVersion` setting must be an exact semantic version; ranges and
tags error (`2025-09`):

```yaml
nodeVersion: 22.20.0
```

`pnpm runtime set <name> <version>` writes to `devEngines.runtime` by default
in pnpm 11.4–11.5. Use `--save-prod`/`-P` for `engines.runtime`
(`11.4-11.5`):

```sh
pnpm runtime set node 24.0.0
pnpm runtime set node 24.0.0 --save-prod
```

`pnpm install --no-runtime`, equivalent to `runtime: false`, skips fetching and
linking lockfile-managed runtimes without removing their entries; frozen
validation still succeeds (`11.1-11.3`). `pnpm outdated` and interactive update
reports include project Node.js, Deno, and Bun `runtime:` dependencies.

Ranges in `devEngines.runtime` and `engines.runtime` are validated for all
three runtimes when `onFail` is `error` or `warn`. Invalid versions report
`ERR_PNPM_BAD_RUNTIME_VERSION` (`11.4-11.5`).

Configure Node.js download mirrors with `nodeDownloadMirrors` in the workspace
file rather than the old `.npmrc` `node-mirror:<channel>` setting:

```yaml
nodeDownloadMirrors:
  release: https://my-mirror.example.com/download/release/
```

## Dependency-declared runtimes

A dependency may declare `engines.runtime`; pnpm installs that Node.js runtime,
binds the dependency CLI to it, and uses it for the dependency's `postinstall`
regardless of the global Node.js version (`2025-11`):

```json
{
  "engines": {
    "runtime": {
      "name": "node",
      "version": "^24.11.0",
      "onFail": "download"
    }
  }
}
```

## Initialization and setup

`pnpm init --init-type=module` creates a manifest with `"type": "module"`
(`2025-05-06`):

```sh
pnpm init --init-type=module
```

`pnpm setup` can configure Nushell. `pnpm init --bare` creates a manifest with
only required fields (`2025-12`).

In pnpm 11, newly initialized projects default to ESM. With
`initPackageManager` enabled, initialization writes `devEngines.packageManager`
instead of `packageManager` (`11.0.0`).

## Script arguments and lifecycle hooks

`pnpm test` forwards everything after `test` directly to the script, like
`pnpm run test`; no separator is needed (`2025-01`):

```sh
pnpm test --watch
```

`pnpm install` runs a project's `preprepare` and `postprepare` scripts starting
in pnpm 10.1.

In pnpm 10, scripts receive only these `npm_package_*` metadata groups:
`name`, `version`, `bin`, `engines`, and `config`. They also receive
`npm_package_json`, the package manifest path, in later pnpm 10 (`2025-04`).

pnpm 11 command scripts write `$ command` to stderr, leaving stdout
pipe-friendly, and show project identity only when executing in another
directory (`11.0.0`). Lifecycle scripts stop receiving config-derived
`npm_config_*` variables, although well-known `npm_*` variables remain.

pnpm 11.6–11.9 restores user-supplied `npm_config_*` variables such as
`npm_config_platform_arch` to lifecycle scripts. It does not restore values
derived from pnpm configuration (`11.6-11.9`).

Hidden scripts whose names start with `.` are omitted from `pnpm run` and may
only be called from other package scripts (`11.0.0`).

## Running multiple scripts

A non-recursive `pnpm run --no-bail` continues through every selected script
but exits nonzero if any fail, matching recursive behavior (`11.6-11.9`).

Use a slash-delimited regular expression to select scripts. Restored
`--sequential`/`-s` support reduces `workspaceConcurrency` to one across and
within projects (`11.10-11.17`):

```sh
pnpm run --sequential "/^build:.*/"
```

## CLI parsing and shortcuts

`pnpm dlx` recognizes its options between `dlx` and the executed command,
including before a `--` separator (`2025-07`).

`install`, `add`, and `dlx` accept `--cpu`, `--libc`, and `--os` to override
configured supported architectures for one command.

In pnpm 11, `-F` means `--filter`. Within `pnpm add`, `-d`, `-p`, `-o`, and
`-e` mean `--save-dev`, `--save-prod`, `--save-optional`, and `--save-exact`
(`11.0.0`).

If a package script is named `clean`, `setup`, `deploy`, or `rebuild`,
`pnpm <name>` runs the script. Use `pnpm pm <name>` for the built-in command.

## Dependency graph inspection

`pnpm why` now puts the searched package at the root and walks reverse
dependency paths back to workspace roots, replacing the older forward tree
(`2026-01-02`). Custom `--find-by` predicates are described in
[configuration-hooks.md](configuration-hooks.md).

Running `pnpm view` without a package name searches upward for the nearest
manifest and queries its package name (`11.6-11.9`).

## Diagnostics

`pnpm doctor` checks installation method, global-bin `PATH`, store and cache
writability, supported filesystem link strategies, registry connectivity, and
an end-to-end offline `file:` install (`11.10-11.17`). It exits nonzero when a
check fails.

```sh
pnpm doctor --json
```

Use `--offline` to skip network checks, `--json` for machine-readable results,
or `--benchmark` to time filesystem and install checks.
