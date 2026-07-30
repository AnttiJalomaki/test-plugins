# Releases and updates

This reference covers native workspace release intent, prerelease lanes,
version bands, dependency-driven changesets, update policy, and workflow-action
updates. These capabilities are attributed to extraction marker
`11.10-11.17`.

## Release intents

`pnpm change` writes intent files compatible with changesets. Preview and
consume them with:

```sh
pnpm change
pnpm change status
pnpm version -r --dry-run
pnpm version from-git
```

Recursive versioning supports dependent propagation, fixed groups, `maxBump`,
filters, dry runs, changelogs, and a committed ledger of consumed intents.

An unpublished package debuts at the version already in its manifest. Pending
intents do not bump that debut; they apply to the next release.

## Release lanes

Move selected packages to a named prerelease lane or back to main:

```sh
pnpm lane next --filter my-package
pnpm lane main --filter my-package
```

A named lane uses `X.Y.Z-<lane>.N`. Configure lanes under `versioning.lanes`.

`versioning.changelog.storage` defaults to `registry`, composing changelogs at
publish time without committing `CHANGELOG.md`. Select `repository` when
committed changelog files are part of the project workflow.

## Epics and major-version bands

`versioning.epics` ties member packages to a lead package. Lead major `M`
restricts members to major versions `M*100` through `M*100+99`. A member cannot
cross the band until the lead advances.

When the lead publishes a stable new major, members rebase to the new band's
floor. Epic membership selectors accept package names, directories, and
negation.

## Changesets from dependency updates

`pnpm update --changeset` creates a patch intent for workspace packages whose
dependencies or optional dependencies changed. A peer-dependency change
creates a major intent. Catalog consumers are included.

Enable the behavior by default and opt out per command when needed:

```yaml
update:
  changeset: true
```

```sh
pnpm update --no-changeset
```

Without `.changeset/config.json`, pnpm warns and does not write an intent.

## GitHub Actions dependencies

`pnpm outdated` and interactive update inspect actions referenced in workflow
files. Non-interactive update requires `--include-github-actions` or workspace
configuration:

```yaml
update:
  githubActions: true
  githubActionsServer: https://github.example.com
```

Updates pin exact commits while preserving release tags in comments.
`update.githubActionsServer` selects an enterprise base URL; otherwise pnpm
uses `GITHUB_SERVER_URL` and then `https://github.com`. Set `githubActions` to
`false` to skip workflow actions in all update modes.

## Current update and audit sections

Top-level `update` and `audit` replace `updateConfig`, `auditConfig`, and
`auditLevel`. Old keys remain accepted until the next major, but new values win
when both are present.

```yaml
update:
  ignoreDeps:
    - webpack
    - "@babel/*"
audit:
  level: high
  ignore:
    - GHSA-xxxx-yyyy-zzzz
```

Use `update.ignoreDeps` for ignored dependency patterns and `audit.level` plus
`audit.ignore` for vulnerability policy.
