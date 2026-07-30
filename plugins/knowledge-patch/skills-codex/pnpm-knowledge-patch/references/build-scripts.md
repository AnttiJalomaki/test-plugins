# Dependency build scripts and approvals

## pnpm 10 default-deny model

Starting with pnpm 10, dependency lifecycle scripts do not run during install
unless the package is approved in `pnpm.onlyBuiltDependencies` (`2025-01`):

```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["fsevents"]
  }
}
```

Set `pnpm.neverBuiltDependencies` to an empty array to recover the former
allow-all behavior. Packages listed in `pnpm.ignoredBuiltDependencies` remain
blocked and do not produce the informational skipped-build message.

Review pending packages with:

```sh
pnpm ignored-builds
pnpm approve-builds
```

When an already-installed dependency is newly added to
`onlyBuiltDependencies`, the next `pnpm install` runs its previously skipped
build script (`2025-12`).

## Command-scoped approvals

Packages directly executed by `pnpm dlx` or `pnpm create` may run their own
postinstall scripts. Their dependencies stay blocked unless named by
`--allow-build`. The same flag on `pnpm add` runs and records named dependencies
in `pnpm.onlyBuiltDependencies` for later installs (`2025-02`):

```sh
pnpm --allow-build=esbuild dlx bundle
pnpm --allow-build=esbuild add bundle
```

In newer pnpm, `--allow-build` persists to the unified `allowBuilds` map
instead (`2026-03`):

```yaml
allowBuilds:
  esbuild: true
```

Use `pnpm approve-builds --global` to review and allow postinstall scripts for
dependencies of globally installed packages. Use `pnpm approve-builds --all`
to approve all pending builds without the interactive selector.

pnpm 11 also accepts explicit package arguments, with `!` for denial
(`11.0.0`):

```sh
pnpm approve-builds esbuild '!core-js'
```

## Fail on unreviewed builds

Enable strict build review to make installation exit nonzero whenever a
dependency has an unreviewed build script:

```ini
strict-dep-builds=true
```

This becomes the default in pnpm 11. Explicit denials still belong in
`allowBuilds` so policy remains reviewable.

## Allow every dependency build

`dangerouslyAllowAllBuilds` permits all dependency build scripts and has the
same effect as an empty `neverBuiltDependencies` list (`2025-04`):

```sh
pnpm config set dangerouslyAllowAllBuilds true
pnpm install --dangerously-allow-all-builds
```

This may be configured globally or for one command. It bypasses individual
review and should only be used when the complete dependency set is trusted.

## Version-scoped approvals

`onlyBuiltDependencies` accepts exact versions and `||` disjunctions
(`2025-10`):

```yaml
onlyBuiltDependencies:
  - nx@21.6.4 || 21.6.5
  - esbuild@0.25.1
```

This limits approval to the listed releases rather than every version of the
package.

## Unified `allowBuilds`

`allowBuilds` replaces `onlyBuiltDependencies` and
`ignoredBuiltDependencies`. Map a package matcher to `true` to permit its
scripts or `false` to deny them; version matchers and `||` disjunctions work
(`2025-12`):

```yaml
allowBuilds:
  esbuild: true
  core-js: false
  nx@21.6.4 || 21.6.5: true
```

pnpm 11 removes `onlyBuiltDependencies`, `onlyBuiltDependenciesFile`,
`neverBuiltDependencies`, `ignoredBuiltDependencies`, and `ignoreDepScripts`.
All approvals and denials must use `allowBuilds`.

## Git dependency builds

Beginning in pnpm 10.26, Git-hosted dependencies cannot run `prepare` during
installation unless allowed through `onlyBuiltDependencies` or `allowBuilds`.
pnpm 10.27 also applies `dangerouslyAllowAllBuilds` to Git dependencies
(`2025-12`).

In current pnpm 11, an `allowBuilds` matcher may be the Git dependency's
repository URL without its resolved commit hash, so approval survives branch
updates (`11.10-11.17`). A package-name-only matcher does not approve a
Git-hosted artifact.
