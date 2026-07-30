# CLI, scripts, and runtimes

This reference covers pnpm version selection, script semantics, package
initialization, recursive execution, runtime provisioning, global packages,
inspection, and diagnostics. Relevant extraction markers include `2025-01`,
`2025-02`, `2025-04`, `2025-05-06`, `2025-07`, `2025-09`, `2025-10`,
`2025-11`, `2025-12`, `2026-01-02`, `migration-10-to-11`, `11.0.0`,
`11.1-11.3`, `11.4-11.5`, `11.6-11.9`, and `11.10-11.17`.

## Package-manager selection

pnpm 10 enables `managePackageManagerVersions` by default and selects its CLI
from the root `packageManager` field:

```json
{
  "packageManager": "pnpm@10.1.0"
}
```

The version cannot have a `v` prefix. Use `pnpm@10.4.0`, not
`pnpm@v10.4.0`. Installing `pnpm` or `@pnpm/exe` via `pnpm add --global`
fails while version management is active; use `pnpm self-update`.

When pnpm explicitly switches to a different CLI, it disables
`managePackageManagerVersions` to avoid another automatic switch. pnpm 11
replaces the related switching/failure settings with `pmOnFail`.

`pnpm with` runs one command under a one-off pnpm version and bypasses the
project's package-manager pin. pnpm 11's self-updater and package-manager
switcher can install the native pnpm 12 build from `next-12`:

```sh
pnpm self-update next-12
```

That installation uses the unscoped `pnpm` package even when upgrading from
`@pnpm/exe`.

## Script argument handling

`pnpm test` passes every argument after `test` to the underlying script, just
like `pnpm run test`; no `--` separator is required:

```sh
pnpm test --watch
```

`pnpm dlx` recognizes its own options between `dlx` and the executed command,
including options before a `--` separator.

## Script selection and failures

Hidden scripts whose names begin with `.` are excluded from `pnpm run` listings
and cannot be invoked directly; other scripts may invoke them.

Use a slash-delimited regular expression to run every matching script. The
restored `--sequential` or `-s` forces `workspaceConcurrency` to one across and
inside packages:

```sh
pnpm run --sequential "/^build:.*/"
```

A non-recursive `pnpm run --no-bail` continues through all matched scripts but
exits nonzero if any fail, matching recursive behavior.

Package scripts named `clean`, `setup`, `deploy`, or `rebuild` shadow built-in
commands in pnpm 11. Force a built-in with `pnpm pm <name>`.

Inside `pnpm add`, short flags mean:

- `-d`: `--save-dev`;
- `-p`: `--save-prod`;
- `-o`: `--save-optional`;
- `-e`: `--save-exact`.

`-F` aliases `--filter` generally.

## Script environment and output

pnpm 10 reduced automatic `npm_package_*` values to `name`, `version`, `bin`,
`engines`, and `config`, and later added `npm_package_json` with the manifest
path.

pnpm 11 prints `$ command` to stderr, keeping script stdout pipe-friendly, and
prints project identity only when running in a different directory. Lifecycle
scripts no longer receive variables derived from pnpm config, although
well-known `npm_*` variables remain.

Later pnpm 11 again preserves `npm_config_*` variables explicitly supplied by
the user, such as `npm_config_platform_arch`. This does not restore variables
automatically derived from configuration.

Reporter output for `pnpm store` and `pnpm config` commands also goes to stderr.

## Recursive workspaces

`pnpm -r pack` packs every workspace project:

```sh
pnpm -r pack
```

The default `workspaceConcurrency` is
`Math.min(os.availableParallelism(), 4)`, so recursive execution uses at most
four concurrent tasks unless configured otherwise.

## Package initialization

pnpm 10 can create an ESM manifest when `initType` is `module` or the command
sets it directly:

```sh
pnpm init --init-type=module
```

`pnpm init --bare` creates a manifest with only required fields. In pnpm 11,
new packages default to `"type": "module"`. With `initPackageManager` enabled,
`pnpm init` writes `devEngines.packageManager` instead of `packageManager`.
That declaration may be a range; pnpm stores the resolved version in the
lockfile and reuses it while compatible.

`pnpm setup` supports Nushell. Run setup after upgrading to pnpm 11 because
global binaries move to `PNPM_HOME/bin`.

## Project runtimes

`devEngines.runtime` can provision Node.js, Deno, or Bun for each workspace
project. Install resolves the requested range, records an exact version and
checksum in the lockfile, and uses the local runtime for scripts. Initially
`onFail` supported only `download`.

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

The legacy `nodeVersion` setting must be an exact semantic version; ranges and
tags error:

```yaml
nodeVersion: 22.20.0
```

In pnpm 11, `pnpm runtime set` replaces `pnpm env use`. It saves development
runtimes by default and production runtimes with `--save-prod` or `-P`:

```sh
pnpm runtime set node 24.0.0
pnpm runtime set node 24.0.0 --save-prod
```

`pnpm install --no-runtime`, equivalent to `runtime: false`, skips downloading
and linking lockfile-managed runtimes without removing their lockfile entries;
frozen validation therefore still passes.

Runtime ranges in `devEngines.runtime` and `engines.runtime` are validated for
Node.js, Deno, and Bun when `onFail` is `warn` or `error`. An invalid resolved
version produces `ERR_PNPM_BAD_RUNTIME_VERSION`.

`pnpm outdated` and interactive update include project runtimes declared with
`runtime:` specifiers.

## Dependency-owned runtimes

A dependency may declare `engines.runtime`. pnpm installs the requested Node.js
version, binds the dependency's CLI to it, and runs that dependency's
`postinstall` under it regardless of the global Node.js version.

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

Publishing may use `publishConfig.engines` to expose different runtime
requirements from those used for development:

```json
{
  "engines": { "node": ">=24" },
  "publishConfig": { "engines": { "node": ">=20" } }
}
```

Configure runtime mirrors in pnpm 11 with `nodeDownloadMirrors`:

```yaml
nodeDownloadMirrors:
  release: https://my-mirror.example.com/download/release/
```

This replaces `node-mirror:<channel>` from `.npmrc`.

## Global package groups

pnpm 11 isolates each `pnpm add -g` group with its own manifest, lockfile, and
`node_modules` under `{pnpmHomeDir}/global/v11/{hash}/`. Removing one package
removes its group; updating creates another group.

Space-separated arguments make independent groups. Comma-separated names make
one shared group whose packages are removed together:

```sh
pnpm add -g foo bar
pnpm add -g foo,bar qar
```

Bare `pnpm install -g`, `pnpm link --global`, and argument-free `pnpm link` are
removed. `pnpm add -g .` replaces the old global-link workflow.

## Inspection and maintenance commands

pnpm 11 adds or changes these commands:

- `pnpm ci` cleans workspace `node_modules` and performs a frozen install.
- `pnpm clean --lockfile` also removes the lockfile.
- `pnpm peers check` reports peer issues from the lockfile.
- `pnpm pack-app` builds Node.js single-executable applications.
- `pnpm sbom` emits CycloneDX 1.7 or SPDX 2.3 JSON.
- `pnpm with` runs a chosen pnpm version once.

`pnpm why` now roots its output at the searched package and walks dependents
back toward workspace roots, reversing the older forward-tree presentation.
Custom `.pnpmfile.cjs` `finders` define predicates for `pnpm list` and
`pnpm why`, selected by `--find-by=<name>`. A finder returns `true` or a string;
the string is also printed as match information.

```js
module.exports = {
  finders: {
    react17: (ctx) =>
      ctx.readManifest().peerDependencies?.react === "^17.0.0",
  },
}
```

```sh
pnpm why --find-by=react17
```

Running `pnpm view` without a package name finds the nearest manifest upward
and queries the package named there.

## Doctor

`pnpm doctor` checks installation method, global-bin `PATH`, store/cache
writability, supported filesystem link strategies, registry connectivity, and
an end-to-end offline `file:` install. It exits nonzero on failures.

```sh
pnpm doctor --json
pnpm doctor --offline
pnpm doctor --benchmark
```

Use `--offline` to omit network checks, `--json` for structured output, and
`--benchmark` for filesystem and install timings.
