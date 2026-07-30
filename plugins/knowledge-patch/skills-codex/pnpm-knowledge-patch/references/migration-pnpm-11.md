# Migrating from pnpm 10 to pnpm 11

## Mechanical migration

Run the migration codemod from the project root (`migration-10-to-11`):

```sh
pnpx codemod run pnpm-v10-to-v11
```

It relocates supported settings and updates the root package-manager declaration,
but the following manual review is required.

## Configuration locations

pnpm 11 does not read the `pnpm` field in `package.json`. `.npmrc` is limited to
authentication and registry settings. Put every other setting in
`pnpm-workspace.yaml` as camelCase. When the codemod finds settings in a
subproject `.npmrc`, it moves them under `packageConfigs["<project-name>"]`.

Replace these three settings:

- `managePackageManagerVersions`
- `packageManagerStrict`
- `packageManagerStrictVersion`

with `pmOnFail`, whose accepted values are `download`, `ignore`, `warn`, and
`error`:

```yaml
pmOnFail: download
```

pnpm 11 ignores `npm_config_*` environment variables used as pnpm
configuration. Rename them to `pnpm_config_*` in CI, shells, containers, and
other launch environments. This is separate from user-defined variables passed
to lifecycle scripts.

Move Node runtime declarations as follows:

- The codemod converts root `useNodeVersion` to `devEngines.runtime`.
- In each subpackage, manually replace
  `package.json#pnpm.executionEnv.nodeVersion` with that subpackage's own
  `devEngines.runtime` declaration.

## Audit and patch migration

`auditConfig.ignoreCves` becomes `auditConfig.ignoreGhsas`. The codemod can
rename the key but cannot convert identifiers. For each CVE, use the matching
GHSA shown in the **More info** column of `pnpm audit`:

```yaml
auditConfig:
  ignoreGhsas:
    - GHSA-xxxx-xxxx-xxxx
```

`ignorePatchFailures` is removed. Every patch application failure throws; fix
the patch or remove the affected dependency.

## Commands removed or reinterpreted

- `pnpm link <package-name>` no longer searches the global store. Pass a
  relative or absolute filesystem path, such as `pnpm link ./foo`.
- Bare `pnpm install -g` is unsupported. Use `pnpm add -g <pkg>`.
- `pnpm server` is removed without a replacement.
- Package scripts named `clean`, `setup`, `deploy`, or `rebuild` shadow the
  built-in commands. Use `pnpm pm <name>` to force the built-in command.

## Runtime and platform requirements

pnpm 11 requires Node.js 22 or newer and is pure ESM. The standalone executable
also requires glibc 2.27 or newer (`11.0.0`).

## Changed installation and security defaults

pnpm 11 defaults to:

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeStrict: false
blockExoticSubdeps: true
strictDepBuilds: true
optimisticRepeatInstall: true
verifyDepsBeforeRun: install
```

Set `minimumReleaseAge: 0` to opt out of the one-day release delay.

The dependency-build settings `onlyBuiltDependencies`,
`onlyBuiltDependenciesFile`, `neverBuiltDependencies`,
`ignoredBuiltDependencies`, and `ignoreDepScripts` are removed. Use
`allowBuilds` for both approvals and denials.

## Registry command implementation

The following registry-facing commands are implemented natively rather than by
passing through to another CLI: `publish`, `view`, `login`, `logout`,
`deprecate`, `unpublish`, `dist-tag`, `version`, `search`, `star`, and `whoami`.

The following commands throw “not implemented”: `access`, `bugs`, `edit`,
`issues`, `owner`, `prefix`, `profile`, `pkg`, `repo`, `set-script`, `team`,
`token`, and `xmas`. Later pnpm 11 minors add native `access` and `team`; check
the installed minor before treating the initial 11.0 result as current.

Publishing reads one-time passwords from `PNPM_CONFIG_OTP`, prompts when needed,
and supports web authentication using a QR code and URL.

## Global installation layout

Each `pnpm add -g` operation creates an isolated group with its own manifest,
lockfile, and `node_modules` under:

```text
{pnpmHomeDir}/global/v11/{hash}/
```

Removing one package removes its entire group; updating creates a new group.
Global binaries moved to `PNPM_HOME/bin`, so run `pnpm setup` after upgrading.
`pnpm link --global` and argument-free `pnpm link` are removed; use
`pnpm add -g .` for the former local-package global-link use case.

## New maintenance and packaging commands

- `pnpm ci` removes workspace `node_modules` and performs a frozen-lockfile
  install.
- `pnpm clean --lockfile` also removes `pnpm-lock.yaml`.
- `pnpm sbom` emits CycloneDX 1.7 or SPDX 2.3 JSON.
- `pnpm peers check` reports lockfile peer issues.
- `pnpm runtime set` replaces `pnpm env use`.
- `pnpm with` runs a one-off pnpm version while bypassing `packageManager` pins.
- `pnpm pack-app` creates Node.js single-executable applications.

`pnpm audit --fix=update` updates vulnerable packages in the lockfile rather
than creating overrides. Add `--interactive` to select advisories. Any
`pnpm audit --fix` adds minimum patched versions to `minimumReleaseAgeExclude`
so the default maturity gate does not delay security fixes.

```sh
pnpm audit --fix=update --interactive
```

## Hooks, store, and script behavior

pnpmfiles may use `.pnpmfile.mjs`. If both `.mjs` and `.pnpmfile.cjs` exist,
only `.mjs` loads.

Store v11 uses one SQLite index at `$STORE/index.db`. Packages missing from the
new index are fetched again on demand rather than discovered from the old
per-package JSON indexes.

Command scripts print `$ command` to stderr, which keeps stdout pipe-friendly,
and show project identity only when running in another directory. Lifecycle
scripts no longer receive config-derived `npm_config_*` values, while
well-known `npm_*` values remain.

With `initPackageManager` enabled, `pnpm init` writes
`devEngines.packageManager` instead of `packageManager`; new packages default
to `"type": "module"`. `devEngines.packageManager` accepts ranges, and the
resolved compatible version is stored and reused from `pnpm-lock.yaml`.

`pnpm approve-builds` accepts positional package names for non-interactive
approval. Prefix a name with `!` to deny it:

```sh
pnpm approve-builds esbuild '!core-js'
```

Script names beginning with `.` are hidden from `pnpm run` and can only be
invoked by another script. `-F` aliases `--filter`. For `pnpm add`, `-d`, `-p`,
`-o`, and `-e` mean `--save-dev`, `--save-prod`, `--save-optional`, and
`--save-exact`.

## Virtual-store-only installs and runtime mirrors

`virtualStoreOnly` populates the virtual store without importer symlinks,
hoisting, binary links, or lifecycle scripts. `pnpm fetch` uses this mode
internally:

```yaml
virtualStoreOnly: true
```

Configure runtime download mirrors in `pnpm-workspace.yaml` with
`nodeDownloadMirrors`; it replaces the `.npmrc` setting
`node-mirror:<channel>`:

```yaml
nodeDownloadMirrors:
  release: https://my-mirror.example.com/download/release/
```
