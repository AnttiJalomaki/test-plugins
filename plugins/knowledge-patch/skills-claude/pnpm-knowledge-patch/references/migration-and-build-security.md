# Migration and dependency-build security

This reference groups migration, dependency lifecycle-script controls, and
patch safety. Relevant extraction markers include `2025-01`, `2025-02`,
`2025-03`, `2025-04`, `2025-10`, `2025-12`, `2026-03`,
`migration-10-to-11`, `11.0.0`, `11.4-11.5`, and `11.10-11.17`.

## Migrate from pnpm 10 to pnpm 11

Run the mechanical codemod at the workspace root:

```sh
pnpx codemod run pnpm-v10-to-v11
```

Review each category after it runs:

- pnpm 11 does not read the `pnpm` field in `package.json`.
- `.npmrc` is restricted to registry and authentication configuration.
- Put all other settings in `pnpm-workspace.yaml` as camelCase keys.
- Settings moved from a subproject `.npmrc` belong under
  `packageConfigs["<project-name>"]`.
- Replace `managePackageManagerVersions`, `packageManagerStrict`, and
  `packageManagerStrictVersion` with `pmOnFail`; accepted values are
  `download`, `ignore`, `warn`, and `error`.
- Replace `auditConfig.ignoreCves` with `auditConfig.ignoreGhsas`. The codemod
  can rename the field but cannot map CVE values; copy the corresponding GHSA
  from the audit report's **More info** column.
- `ignorePatchFailures` is removed. Every patch failure throws, so repair the
  patch or remove the affected dependency.
- Convert root `useNodeVersion` to `devEngines.runtime`. In subpackages,
  manually replace `pnpm.executionEnv.nodeVersion` with that package's own
  `devEngines.runtime` declaration.
- Rename configuration environment variables from `npm_config_*` to
  `pnpm_config_*` in CI, shells, and images. This does not prohibit the
  user-supplied `npm_config_*` variables later restored to lifecycle scripts.
- `pnpm link` needs a relative or absolute path; it no longer resolves a name
  through the global store.
- Bare `pnpm install -g` is removed. Use `pnpm add -g <package>`.
- `pnpm server` is removed without replacement.
- Scripts named `clean`, `setup`, `deploy`, or `rebuild` shadow built-ins. Use
  `pnpm pm <name>` when the built-in is intended.

## Runtime and platform prerequisites

pnpm 11 requires Node.js 22 or newer and is distributed as pure ESM. The
standalone executable requires glibc 2.27 or newer. Check CI base images and
deployment hosts before changing the package-manager declaration.

## Installation-policy defaults in pnpm 11

pnpm 11 defaults to:

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeStrict: false
blockExoticSubdeps: true
strictDepBuilds: true
optimisticRepeatInstall: true
verifyDepsBeforeRun: install
```

Set `minimumReleaseAge: 0` only when intentionally opting out of the default
one-day delay. Legacy `onlyBuiltDependencies`, `onlyBuiltDependenciesFile`,
`neverBuiltDependencies`, `ignoredBuiltDependencies`, and `ignoreDepScripts`
are unsupported; use `allowBuilds`.

## Build controls in pnpm 10

pnpm 10 blocks dependency lifecycle scripts by default. Permit reviewed
packages with `pnpm.onlyBuiltDependencies`:

```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["fsevents"]
  }
}
```

An empty `pnpm.neverBuiltDependencies` restores the older allow-all behavior.
`pnpm.ignoredBuiltDependencies` blocks named packages without emitting the
informational skipped-build message. `dangerouslyAllowAllBuilds` also allows
every dependency script, globally or for one invocation:

```sh
pnpm config set dangerouslyAllowAllBuilds true
pnpm install --dangerously-allow-all-builds
```

Use these broad exceptions sparingly. `strict-dep-builds=true` makes install
fail when any dependency has an unreviewed build script.

Review pending work with:

```sh
pnpm ignored-builds
pnpm approve-builds
pnpm approve-builds --global
```

The global form reviews dependencies of globally installed packages. Adding
an already-installed package to `onlyBuiltDependencies` causes the next
install to run its previously skipped script.

## One-command and persisted approval

`--allow-build=<package>` permits selected dependency scripts for `add`,
`dlx`, and `create` workflows. A `dlx` or `create` target may run its own
postinstall by default, but its dependencies remain blocked unless named.

```sh
pnpm --allow-build=esbuild dlx bundle
pnpm --allow-build=esbuild add bundle
```

Earlier pnpm 10 writes these approvals into `onlyBuiltDependencies`; newer
pnpm writes them into `allowBuilds`. `pnpm approve-builds --all` accepts every
pending build without an interactive selector.

pnpm 11 also accepts package arguments and explicit denial:

```sh
pnpm approve-builds esbuild '!core-js'
```

## Unified `allowBuilds`

`allowBuilds` is the preferred map for both approval and denial. Matchers may
include exact versions and `||` disjunctions:

```yaml
allowBuilds:
  esbuild: true
  core-js: false
  "nx@21.6.4 || 21.6.5": true
```

The older `onlyBuiltDependencies` also gained exact-version and `||` matchers
before being removed in pnpm 11.

Git-hosted dependencies cannot run `prepare` unless approved. pnpm 10.27 also
honors `dangerouslyAllowAllBuilds` for them. In newer pnpm 11, an
`allowBuilds` entry may match the repository URL without the resolved commit,
so branch updates remain approved. A package-name-only matcher does not approve
a Git artifact.

## Project lifecycle scripts

pnpm 10.1 added `preprepare` and `postprepare` around a project's `prepare`
script. These are project scripts, distinct from blocked dependency build
scripts.

## Dependency patches

`patchedDependencies` supports package name, range, and exact-version keys.
Exact keys win over ranges, which win over name-only keys. Do not overlap
ranges.

```yaml
patchedDependencies:
  foo: patches/foo-1.patch
  "foo@^2.0.0": patches/foo-2.patch
  foo@2.1.0: patches/foo-3.patch
```

`foo@*` matches like a name-only key, except application failure is not
ignored. In pnpm 10, the tri-state `ignorePatchFailures` policy behaves as
follows:

- unset: exact/range failures are errors; name-only failures are ignored;
- `false`: every failure is an error;
- `true`: every failure is a warning.

`pnpm.allowNonAppliedPatches` was renamed to `pnpm.allowUnusedPatches`; the old
name warns but remains supported in pnpm 10. pnpm 11 removes
`ignorePatchFailures` and makes all failures fatal.

Patch files are rejected if `diff --git` headers attempt to write, delete, or
rename outside the patched package directory. Dependency aliases containing
path-traversal segments are rejected while reading manifests and while linking
`node_modules`.
