# Workspace release and update management

## Changes and versioning

pnpm 11.10–11.17 includes native changesets-compatible release management
(`11.10-11.17`):

```sh
pnpm change
pnpm change status
pnpm version -r --dry-run
pnpm version from-git
```

- `pnpm change` writes release-intent files compatible with changesets.
- `pnpm change status` previews the release plan.
- `pnpm version -r` consumes intents with dependent propagation, fixed groups,
  `maxBump`, filtering, dry runs, changelog generation, and a committed
  consumption ledger.
- `pnpm version from-git` derives versions from Git state.

An unpublished package debuts at the version already in its manifest. Pending
intents do not apply until its next release.

## Prerelease lanes

Move packages onto a named prerelease lane or back to the main line:

```sh
pnpm lane next --filter my-package
pnpm lane main --filter my-package
```

`pnpm lane <name>` assigns versions in the form `X.Y.Z-<lane>.N`. Configuration
lives under `versioning.lanes`.

`versioning.changelog.storage` defaults to `registry`, which composes
changelogs during publish without committing `CHANGELOG.md`. Select
`repository` when changelog files should be committed.

## Epics and major-version bands

`versioning.epics` associates member packages with a lead package. If the lead
major is `M`, members are limited to majors `M*100` through `M*100+99`. A
member cannot cross the band until the lead advances.

Publishing a stable lead-major release rebases members to the new band's floor.
Epic membership accepts package-name, directory, and negated selectors.

## Convergence overrides

An empty-range override selector such as `"pkg@"` changes only dependency
edges whose declared ranges accept the exact override value. This converges
compatible consumers without forcing incompatible ones:

```yaml
overrides:
  "form-data@": 4.0.6
```

The override value must be exact. pnpm warns when all declared ranges accept a
newer possible convergence target.

## Generate intents from dependency updates

`pnpm update --changeset` writes release intents for workspace packages whose
dependencies change:

- Dependency or optional-dependency changes produce a patch intent.
- Peer-dependency changes produce a major intent.
- Catalog consumers are included.

Enable this by default and opt out per command when needed:

```yaml
update:
  changeset: true
```

```sh
pnpm update --no-changeset
```

Without `.changeset/config.json`, pnpm warns and writes no intent.

## Update workflow actions

`pnpm outdated` and interactive `pnpm update` inspect dependencies in workflow
files. Non-interactive updates require `--include-github-actions` or
`update.githubActions: true`.

Updates pin exact commit SHAs and preserve release tags in comments.
`update.githubActionsServer` selects a GitHub Enterprise base URL; otherwise
pnpm uses `GITHUB_SERVER_URL`, then `https://github.com`. Setting
`githubActions: false` disables workflow-action inspection everywhere.

```yaml
update:
  githubActions: true
  githubActionsServer: https://github.example.com
```

## Current update configuration

Top-level `update` replaces `updateConfig` in pnpm 11.10–11.17. The deprecated
form remains until the next major, but the new section wins if both appear.
Use `update.ignoreDeps` for dependency-name patterns:

```yaml
update:
  ignoreDeps:
    - webpack
    - "@babel/*"
```

Top-level `audit` similarly replaces `auditConfig` and `auditLevel`; see
[security-audit-sbom.md](security-audit-sbom.md).

## Staged and atomic publication

Use `pnpm stage` to keep a release hidden until approval, or
`pnpm publish --recursive --batch` for registry-supported all-or-nothing
workspace publication. Command details and authentication behavior are in
[registries-auth-publishing.md](registries-auth-publishing.md).
