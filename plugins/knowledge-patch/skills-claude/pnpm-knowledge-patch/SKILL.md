---
name: pnpm-knowledge-patch
description: pnpm
version: 11.17.0
license: MIT
metadata:
  author: Nevaberry
---


# pnpm Knowledge Patch

Use this skill when maintaining pnpm workspaces, migrating pnpm major versions,
configuring installs or registries, approving dependency builds, publishing
packages, managing runtimes, or diagnosing lockfile and store behavior.

First inspect the repository's `package.json`, `pnpm-workspace.yaml`, `.npmrc`,
and `pnpm-lock.yaml`. Respect the pinned pnpm version and apply only behavior
available to that version. Repository configuration, lockfiles, and observed
command behavior take precedence over generalized guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-and-build-security.md](references/migration-and-build-security.md) | pnpm 10-to-11 migration, dependency build approvals, patch safety, install policy |
| [references/configuration-and-workspaces.md](references/configuration-and-workspaces.md) | configuration locations, workspaces, catalogs, deploy, linking, hoisting, peers |
| [references/installation-lockfiles-and-store.md](references/installation-lockfiles-and-store.md) | installs, lockfiles, integrity, virtual stores, pacquet, platform selection |
| [references/cli-scripts-and-runtimes.md](references/cli-scripts-and-runtimes.md) | scripts, runtime provisioning, initialization, inspection, diagnostics, globals |
| [references/registries-publishing-and-supply-chain.md](references/registries-publishing-and-supply-chain.md) | authentication, registries, audit, trust, publishing, SBOMs |
| [references/releases-and-updates.md](references/releases-and-updates.md) | native release management, lanes, update policy, changesets, GitHub Actions |

## Start with the pinned version

1. Read the root `packageManager` or `devEngines.packageManager` declaration.
2. Check the lockfile format and the actual CLI with `pnpm --version`.
3. Do not write pnpm 11 configuration into locations that pnpm 11 ignores.
4. When a repository is still on pnpm 10, do not remove a compatibility control
   merely because pnpm 11 has replaced it.
5. After configuration changes, use a frozen install in CI and a dry run where
   supported before allowing writes.

## pnpm 11 migration essentials

Run the migration codemod from the workspace root:

```sh
pnpx codemod run pnpm-v10-to-v11
```

Then review its output manually:

- Put non-registry settings in `pnpm-workspace.yaml` with camelCase names.
- Keep `.npmrc` for registry and authentication settings only.
- Remove the ignored `pnpm` object from `package.json` after its settings move.
- Replace package-manager strictness and switching controls with `pmOnFail`.
- Rename configuration environment variables from `npm_config_*` to
  `pnpm_config_*`; user-created lifecycle variables are a separate case.
- Replace legacy dependency-build lists with the `allowBuilds` map.
- Convert audit exclusions from CVEs to GHSAs manually.
- Move project runtime declarations to `devEngines.runtime`.
- Expect patch application failures to be fatal.

pnpm 11 requires Node.js 22 or newer, is pure ESM, and requires glibc 2.27 or
newer for the standalone executable. Its security-oriented defaults include a
one-day release-age gate, blocking exotic transitive dependencies, strict
dependency-build review, optimistic repeat installs, and dependency freshness
verification before scripts.

See [references/migration-and-build-security.md](references/migration-and-build-security.md)
for removed commands, renamed controls, and staged build-policy changes.

## Dependency build scripts

Treat dependency scripts as explicitly reviewed capabilities. The durable
pnpm 11 representation is a boolean matcher map:

```yaml
allowBuilds:
  esbuild: true
  core-js: false
  "nx@21.6.4 || 21.6.5": true
```

Useful approval flows include:

```sh
pnpm ignored-builds
pnpm approve-builds
pnpm approve-builds --all
pnpm approve-builds esbuild '!core-js'
pnpm --allow-build=esbuild add bundle
```

In pnpm 10, `onlyBuiltDependencies`, `ignoredBuiltDependencies`,
`neverBuiltDependencies`, `strict-dep-builds`, and
`dangerouslyAllowAllBuilds` may still be relevant. Git dependencies need
explicit approval for `prepare`; newer pnpm can approve them by repository URL.

## Configuration placement

For pnpm 11, make `pnpm-workspace.yaml` authoritative:

```yaml
pmOnFail: download
verifyDepsBeforeRun: install
allowBuilds:
  esbuild: true
minimumReleaseAge: 1440
```

Workspace configuration accepts camelCase setting names. Use project-specific
`packageConfigs` after migrating settings from subproject `.npmrc` files.
Registry URLs, credentials, certificates, and token helpers remain in auth
configuration rather than workspace policy.

Use structured config paths when editing nested values:

```sh
pnpm config get 'packageExtensions["@babel/parser"].peerDependencies["@babel/types"]'
pnpm config get --json catalog
pnpm config set .ignoreScripts true
```

pnpmfiles can be CommonJS or ESM; `.pnpmfile.mjs` wins when both formats exist.
Multiple configured pnpmfiles and config-dependency plugins load hooks, so make
their order explicit when hooks interact.

## Lockfiles and reproducible installs

Keep integrity failures fatal unless the exact registry change has been
reviewed. On pnpm 10.34+, accept a changed tarball only with the narrow option:

```sh
pnpm install --update-checksums
```

`--force`, `update`, and `--fix-lockfile` do not bypass a checksum mismatch.
Missing tarball integrity is also fatal in pnpm 11, except for Git and local
`file:` tarballs. Git resolutions must contain a full 40-character commit SHA.

For CI, prefer:

```sh
pnpm install --frozen-lockfile
```

An incompatible lockfile is fatal in frozen CI installs. pnpm 10.33 can read
pnpm 11's two-document lockfile, and pnpm 11 can regenerate a missing root
lockfile from a compatible installed snapshot only in a non-frozen install.

Preview resolution without writes when supported:

```sh
pnpm install --dry-run
```

## Catalogs, workspace dependencies, and deploy

Prefer catalog specifiers for centrally governed workspace versions:

```yaml
catalogMode: strict
cleanupUnusedCatalogs: true
```

```sh
pnpm add --save-catalog lodash
pnpm add --save-catalog-name=testing vitest
pnpm dlx shx@catalog:
```

`pnpm update` updates catalog entries. A bare `workspace:` means
`workspace:*` and is replaced with the concrete version when publishing.
Current deploy supports catalog-managed dependencies. pnpm 10 deployment
requires injected workspace packages; deployment always uses a local virtual
store even if the source workspace enables the global virtual store.

## Release-age and trust policy

The central release-age control is measured in minutes:

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeExclude:
  - "webpack@4.47.0 || 5.102.1"
trustPolicy: no-downgrade
trustPolicyExclude:
  - chokidar@4.0.3
```

Exact requests remain gated unless explicitly excluded. `outdated` follows the
same age policy, and pnpm may select the highest mature release across a major
boundary. `trustLockfile: true` is appropriate only when lockfile changes pass
a trusted review. Staged-publish evidence ranks above trusted-publisher and
provenance evidence; highest-rank trusted-publisher evidence requires
provenance.

## Runtime provisioning

Use `devEngines.runtime` for project development runtimes and `engines.runtime`
for dependency or production runtime needs:

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

```sh
pnpm runtime set node 24.0.0
pnpm runtime set node 24.0.0 --save-prod
pnpm install --no-runtime
```

Exact lockfile-resolved runtimes can be Node.js, Deno, or Bun. Warning and
error modes validate requested ranges. Configure runtime download mirrors with
`nodeDownloadMirrors`, not the removed `.npmrc` channel setting.

## Verification checklist

- Confirm the CLI version matches the repository declaration and lockfile.
- Validate configuration is in a location read by that pnpm major.
- Review every `allowBuilds: true` entry and every release-age/trust exception.
- Run a frozen install in CI and treat integrity failures as review events.
- Check runtime, registry, proxy, and credential behavior in the actual CI
  environment rather than assuming local shell behavior matches.
- Dry-run install, pack, audit-fix, versioning, or update operations where the
  command supports it.
- Use `pnpm doctor --json` when installation, PATH, filesystem links, store,
  cache, or registry connectivity is unclear.
