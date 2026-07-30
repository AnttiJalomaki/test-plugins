---
name: pnpm-knowledge-patch
description: pnpm
version: 11.17.0
license: MIT
metadata:
  author: Nevaberry
---


# pnpm Knowledge Patch

Use this skill when maintaining pnpm workspaces, installs, lockfiles, package
scripts, registries, publishing, or release automation. Start by identifying the
project's pnpm major from `packageManager`, `devEngines.packageManager`, the
lockfile, or CI setup. Apply version-specific guidance only to that major.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-pnpm-11.md](references/migration-pnpm-11.md) | pnpm 10-to-11 codemod, breaking removals, configuration relocation, new defaults |
| [build-scripts.md](references/build-scripts.md) | Dependency build approval, `allowBuilds`, strict failures, Git builds |
| [configuration-hooks.md](references/configuration-hooks.md) | `pnpm-workspace.yaml`, config dependencies, patches, pnpmfiles, custom hooks |
| [installs-lockfiles-store.md](references/installs-lockfiles-store.md) | Install semantics, lockfile integrity, stores, linkers, pacquet, dry runs |
| [workspaces-catalogs-deploy.md](references/workspaces-catalogs-deploy.md) | Catalogs, workspace linking, injected dependencies, deploy, recursive execution |
| [runtimes-scripts-cli.md](references/runtimes-scripts-cli.md) | Runtime provisioning, script behavior, initialization, diagnostics, CLI details |
| [registries-auth-publishing.md](references/registries-auth-publishing.md) | Registry selection, credentials, TLS, package formats, global installs, publishing |
| [security-audit-sbom.md](references/security-audit-sbom.md) | Release-age and trust policies, audit, signatures, SBOMs, dependency hardening |
| [release-management.md](references/release-management.md) | Changes, versions, lanes, epics, staged releases, update automation |

## First: identify the configuration era

For pnpm 10, settings may live in the `pnpm` field of `package.json`, `.npmrc`,
or `pnpm-workspace.yaml`, depending on the pnpm 10 minor. Prefer camelCase
workspace-file settings for an easier migration.

For pnpm 11, use these authoritative locations:

- Put authentication and registry settings in `.npmrc`.
- Put every other pnpm setting in `pnpm-workspace.yaml` with camelCase keys.
- Put per-project migrated settings under `packageConfigs["<project-name>"]`.
- Replace `npm_config_*` configuration variables with `pnpm_config_*`.
- Use `pmOnFail` instead of the three package-manager strictness settings.
- Use `allowBuilds` instead of all legacy dependency-build allow/deny settings.

Do not copy a pnpm 11 setting back into a pnpm 10-only project without checking
that the installed minor recognizes it.

## pnpm 11 migration quick reference

Run the mechanical migration at the workspace root:

```sh
pnpx codemod run pnpm-v10-to-v11
```

Then complete the changes the codemod cannot safely infer:

1. Require Node.js 22 or newer. The standalone executable also needs glibc
   2.27 or newer, and pnpm itself is pure ESM.
2. Move non-registry settings to `pnpm-workspace.yaml`; remove reliance on the
   root `package.json#pnpm` field.
3. Translate every `auditConfig.ignoreCves` entry to its GHSA identifier under
   `auditConfig.ignoreGhsas`; a key rename alone is insufficient.
4. Replace `package.json#pnpm.executionEnv.nodeVersion` in each subpackage with
   that package's `devEngines.runtime` declaration.
5. Replace `managePackageManagerVersions`, `packageManagerStrict`, and
   `packageManagerStrictVersion` with `pmOnFail`.
6. Remove `ignorePatchFailures`; patch application failures are always fatal.
7. Convert dependency-build permissions to `allowBuilds`.
8. Pass a filesystem path to `pnpm link`; package-name lookup in the global
   store is gone.
9. Replace argument-free `pnpm install -g`; install named packages with
   `pnpm add -g <pkg>`.
10. Remove `pnpm server` usage; it has no replacement.
11. If a package script is named `clean`, `setup`, `deploy`, or `rebuild`, use
    `pnpm pm <name>` when the built-in command is intended.
12. Review registry commands: several are now native, while unsupported npm
    passthrough commands fail with “not implemented.”
13. Run `pnpm setup` so global binaries resolve from `PNPM_HOME/bin`.

Also account for the new install and security defaults. Notable defaults are
`minimumReleaseAge: 1440`, `blockExoticSubdeps: true`,
`strictDepBuilds: true`, `optimisticRepeatInstall: true`, and
`verifyDepsBeforeRun: install`. Set `minimumReleaseAge: 0` only when the
one-day maturity delay is intentionally unwanted.

See [migration-pnpm-11.md](references/migration-pnpm-11.md) for command
removals, global-install isolation, store migration, initialization, script
environment changes, and the new maintenance commands.

## Dependency build permissions

pnpm 10 blocks unapproved dependency lifecycle scripts. Earlier pnpm 10 minors
use `onlyBuiltDependencies` for allowed packages and
`ignoredBuiltDependencies` for explicit silent denials. An empty
`neverBuiltDependencies` or `dangerouslyAllowAllBuilds` enables all builds and
should be reserved for deliberately trusted dependency sets.

The unified form is:

```yaml
allowBuilds:
  esbuild: true
  core-js: false
  nx@21.6.4 || 21.6.5: true
```

In pnpm 11, this map is mandatory; the legacy controls are removed. Useful
review commands include:

```sh
pnpm ignored-builds
pnpm approve-builds
pnpm approve-builds --all
pnpm approve-builds esbuild '!core-js'
```

Use `strictDepBuilds` to fail when a dependency has an unreviewed build. A Git
dependency needs explicit approval too; current pnpm can approve its repository
URL so the permission survives commit changes. See
[build-scripts.md](references/build-scripts.md) for command-scoped approvals,
global approvals, version matchers, and the exact Git behavior.

## Safe installation and lockfile handling

Treat a locked tarball checksum mismatch as a supply-chain event. A normal
install must not silently overwrite the checksum. Use the narrow opt-in only
after validating why registry content changed:

```sh
pnpm install --update-checksums
```

`--force`, `pnpm update`, and `--fix-lockfile` do not bypass this check. Missing
tarball integrity is also fatal in pnpm 11 except for Git-hosted and local
`file:` tarballs.

Preview dependency resolution without filesystem changes:

```sh
pnpm install --dry-run
```

For a hermetic read-only cache, combine a populated store with offline and
frozen modes:

```sh
pnpm install --frozen-store --offline --frozen-lockfile
```

In CI, an incompatible lockfile is fatal under frozen mode. A symlinked
`pnpm-lock.yaml` may be read only when the install does not need to change it.
Use [installs-lockfiles-store.md](references/installs-lockfiles-store.md) for
lockfile migrations, virtual stores, hoisting, package maps, peer deduplication,
and recovery from the installed lockfile snapshot.

## Catalog and workspace essentials

Use `catalog:` or `catalog:<name>` to centralize dependency versions. `pnpm add`
can select a matching default catalog entry automatically, and these commands
write catalog entries explicitly:

```sh
pnpm add --save-catalog lodash
pnpm add --save-catalog-name=testing vitest
```

Choose `catalogMode: strict` to reject requests outside the catalog, `prefer`
to use compatible entries and otherwise fall back, or `manual` for no automatic
catalog insertion. Use `cleanupUnusedCatalogs: true` only when install-time
removal of unused entries is desired.

For deployment, injected local workspace dependencies are hard-linked rather
than symlinked. pnpm 10 requires `injectWorkspacePackages: true` for
`pnpm deploy`; current deployment also supports catalog-managed workspaces and
always creates a deployment-local virtual store.

See [workspaces-catalogs-deploy.md](references/workspaces-catalogs-deploy.md)
for link overrides, injected-file synchronization, concurrency, bare
`workspace:` specifiers, `dlx` catalogs, and deployment lockfile behavior.

## Runtimes and scripts

`devEngines.runtime` can lock Node.js, Deno, or Bun to an exact resolved version
and checksum in `pnpm-lock.yaml`. pnpm-managed scripts then run with that local
runtime. Use `pnpm runtime set` for project runtime declarations and
`pnpm install --no-runtime` to skip fetching them without deleting lockfile
entries.

```sh
pnpm runtime set node 24.0.0
pnpm runtime set node 24.0.0 --save-prod
```

Dependency CLIs can similarly declare `engines.runtime`. Validate runtime ranges
when `onFail` is `warn` or `error`; invalid versions are not silently ignored.

Script automation should account for these details:

- `pnpm test` forwards trailing arguments without a `--` separator.
- `pnpm run --no-bail` continues but exits nonzero after any failure.
- Slash-delimited regular expressions select multiple scripts.
- `--sequential` or `-s` forces workspace concurrency to one.
- Script command banners and reporter progress use stderr where documented, so
  stdout remains suitable for pipelines.

See [runtimes-scripts-cli.md](references/runtimes-scripts-cli.md) for hidden
scripts, environment variables, `pnpm why`, `pnpm doctor`, initialization,
architecture overrides, and package-manager version selection.

## Registry, audit, and release operations

Keep credentials URL-scoped. For multiple scopes on one registry, use
scope-qualified token keys. For file-free CI configuration, pass URL-scoped
`pnpm_config_//...` variables through an environment facility that permits
punctuation in names. Prefer the structured global `_auth` setting when a
single trusted value must carry registry-wide and scope-specific credentials.

Before publishing or promoting packages:

- Use `pnpm audit signatures` to verify installed registry signatures.
- Use `pnpm sbom` for CycloneDX or SPDX output; select, split, or exclude
  peer-only trees as needed.
- Use `pnpm pack --dry-run` to inspect the tarball file list.
- Use staged publishing when a release must remain hidden until approval.
- Use atomic recursive batch publishing only with a registry that implements
  pnpm's batch endpoint.

For native workspace releases, use `pnpm change`, `pnpm change status`, and
`pnpm version -r`; release lanes, epics, and dependency-update intents extend
that workflow. See [registries-auth-publishing.md](references/registries-auth-publishing.md),
[security-audit-sbom.md](references/security-audit-sbom.md), and
[release-management.md](references/release-management.md) for the complete
behavior and configuration.
