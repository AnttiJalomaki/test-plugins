# Dependency security, audit, trust, and SBOMs

## Delay newly published versions

`minimumReleaseAge` delays installation until a version has existed for the
configured number of minutes (`2025-09`):

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeExclude:
  - webpack
  - "@eslint/*"
```

`minimumReleaseAgeExclude` accepts package names and, starting in pnpm 10.17,
package patterns. Exact-version requests remain gated even when metadata is
cached. If a stable dist-tag points to a too-new version, pnpm does not fall
back to a prerelease.

Later pnpm 10 accepts exact versions and `||` disjunctions, allowing selected
releases to bypass the delay without exempting the package (`2025-10`):

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeExclude:
  - nx@21.6.5
  - webpack@4.47.0 || 5.102.1
```

`pnpm outdated` respects the maturity policy. When `latest` is too new, pnpm
selects the highest sufficiently mature version even across major-version
boundaries.

pnpm 11 defaults `minimumReleaseAge` to 1440 minutes and
`minimumReleaseAgeStrict` to `false` (`11.0.0`). Set age to 0 to opt out.

## Trust downgrade policy

`trustPolicy: no-downgrade` rejects a release whose trust evidence is weaker
than an earlier release (`2025-11`):

```yaml
trustPolicy: no-downgrade
trustPolicyExclude:
  - chokidar@4.0.3
  - webpack@4.47.0 || 5.102.1
```

`trustPolicyExclude` allows named versions. Policy failures stay fatal for
optional dependencies, and prerelease trust evidence is ignored while
installing a stable release.

`trustPolicyIgnoreAfter` skips trust-policy checks for versions published more
than the configured age ago (`2025-12`).

`trustLockfile: true` skips both `minimumReleaseAge` and no-downgrade checks for
entries loaded from an already-reviewed lockfile (`11.1-11.3`). It defaults to
`false` and is intended only when lockfile changes pass a trusted review
process.

```yaml
trustLockfile: true
```

Trusted-publisher evidence receives its highest trust rank only when provenance
is also present (`11.4-11.5`). Registry metadata containing an `approver` field
counts as staged-publish evidence and ranks above trusted publishers and
provenance attestations.

## Block risky dependency sources

`blockExoticSubdeps` rejects exotic transitive sources such as `git+ssh:` and
direct HTTPS tarballs, while direct dependencies may still use them
(`2025-12`):

```yaml
blockExoticSubdeps: true
```

This is enabled by default in pnpm 11. Lockfile integrity and Git validation are
covered in [installs-lockfiles-store.md](installs-lockfiles-store.md).

## Audit filtering

pnpm 10 adds two forms of audit exclusion (`2025-05-06`):

```sh
pnpm audit --ignore-unfixable
pnpm audit --ignore=CVE-2021-1234 --ignore=CVE-2021-5678
```

`--ignore-unfixable` hides vulnerabilities without a fix. Repeatable
`--ignore` hides selected CVEs even when fixes exist.

For pnpm 11 migration, replace `auditConfig.ignoreCves` with
`auditConfig.ignoreGhsas` and manually translate each CVE to the corresponding
GHSA from the audit report (`migration-10-to-11`).

## Audit fixes

pnpm 11 supports `pnpm audit --fix=update`, which updates vulnerable packages
in the lockfile rather than adding overrides. `--interactive` selects
advisories (`11.0.0`):

```sh
pnpm audit --fix=update --interactive
```

Every form of `pnpm audit --fix` adds the minimum patched releases to
`minimumReleaseAgeExclude`, preventing the default age gate from delaying a
security fix.

## Signature verification

`pnpm audit signatures` verifies installed packages against ECDSA registry
signatures and keys published at `/-/npm/v1/keys` (`11.1-11.3`). It honors
scoped registries and skips registries that publish no signing keys.

```sh
pnpm audit signatures
```

## Generate SBOMs

pnpm 11 introduces `pnpm sbom` with CycloneDX 1.7 and SPDX 2.3 JSON output
(`11.0.0`). For CycloneDX, `--sbom-spec-version` selects 1.5, 1.6, or 1.7; the
default is 1.7, and the option is invalid for other formats (`11.1-11.3`):

```sh
pnpm sbom --sbom-format cyclonedx --sbom-spec-version 1.6
```

`--out` writes one output file and `--split` creates one SBOM for every selected
workspace package (`11.6-11.9`):

```sh
pnpm sbom --out sbom.json
pnpm sbom --split
```

When exactly one filtered package is selected, it becomes the root component.
CycloneDX marks components reachable only through development dependencies as
excluded development components.

Use `--exclude-peers` to omit peer dependencies and transitive subtrees
reachable only through those peers:

```sh
pnpm sbom --exclude-peers
```

## Current audit configuration

Top-level `audit` replaces `auditConfig` and `auditLevel` in pnpm 11.10–11.17
(`11.10-11.17`). Deprecated keys continue until the next major, but the new
form wins when both are present:

```yaml
audit:
  level: high
  ignore:
    - GHSA-xxxx-yyyy-zzzz
```
