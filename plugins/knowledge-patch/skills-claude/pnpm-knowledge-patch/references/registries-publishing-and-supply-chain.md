# Registries, publishing, and supply-chain policy

This reference covers registry selection and credentials, release-age and
trust controls, audit, packing, publishing, and SBOM generation. Relevant
extraction markers include `2025-02`, `2025-05-06`, `2025-09`, `2025-10`,
`2025-11`, `2025-12`, `2026-01-02`, `11.0.0`, `11.1-11.3`, `10.34.0`,
`11.4-11.5`, `11.6-11.9`, and `11.10-11.17`.

## Scoped and named registries

Pass a scoped registry directly; `--config.` is no longer needed:

```sh
pnpm --@scope:registry=https://scope.example.com/npm install
```

pnpm 11 dependencies can use the built-in `gh:` prefix for GitHub Packages or
a name declared in `namedRegistries`. Authentication continues to use the
normal credentials for the mapped URL.

```yaml
namedRegistries:
  work: https://npm.work.example.com/
```

```sh
pnpm add work:@corp/lib@^2.0.0
```

The `gh` name can be remapped for GitHub Enterprise Server.

`pnpm login --scope <scope>` writes a normalized `@scope:registry` mapping and
the token into pnpm's auth file. A missing leading `@` is supplied:

```sh
pnpm login --scope acme
```

## URL-scoped credentials

Prefer credentials attached to their destination URL. pnpm 10.25 recognizes
inline `cert`, `ca`, and `key` values as well as `certfile`, `cafile`, and
`keyfile`:

```ini
//registry.example.com/:ca=-----BEGIN CERTIFICATE-----...
```

From pnpm 10.34, unscoped `_authToken`, `_auth`, `username`/`_password`,
`tokenHelper`, and inline certificate/key values are pinned at load time to the
registry declared in the same config source. A later `registry=` cannot redirect
them. A deprecation warning identifies the source and pinned URL.

Do not put environment expansion in `tokenHelper` or registry-scoped
`<url>:tokenHelper`; pnpm 10.27+ rejects it.

## File-free CI authentication

pnpm 11 can read URL-scoped values from arbitrary-name `npm_config_//…` or
`pnpm_config_//…` variables. Shell identifiers cannot contain these
characters, so pass them through `env` or a capable CI secret mechanism:

```sh
env "pnpm_config_//registry.npmjs.org/:_authToken=$NPM_TOKEN" pnpm install
```

These trusted variables override project/workspace `.npmrc` entries but not
CLI options. The `pnpm_config_` form wins over `npm_config_`.

When several package scopes share a host, include a scope after the registry
URL. An unscoped token remains the fallback:

```ini
@org-a:registry=https://npm.pkg.github.com/
@org-b:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:@org-a:_authToken=ORG_A_TOKEN
//npm.pkg.github.com/:@org-b:_authToken=ORG_B_TOKEN
//npm.pkg.github.com/:_authToken=FALLBACK_TOKEN
```

## Structured authentication

Current pnpm supports `_auth` as a structured map containing registry-wide
(`@`) and scope-specific credentials together with their URL. It is accepted
only from the global `config.yaml` or `pnpm_config__auth`, never from a project
file. The global config also accepts `registries` and `namedRegistries`.

```sh
export pnpm_config__auth='{"https://registry.npmjs.org":{"@":{"authToken":"npm-token"},"@org":{"authToken":"org-token"}}}'
```

## Release-age policy

`minimumReleaseAge` delays a version for a number of minutes after publication:

```yaml
minimumReleaseAge: 1440
minimumReleaseAgeExclude:
  - webpack
  - "@eslint/*"
```

Important semantics:

- Exact-version requests remain gated, including with cached metadata.
- A too-new stable dist-tag target does not trigger fallback to a prerelease.
- Package patterns are supported in exclusions from pnpm 10.17.
- Exact versions and `||` disjunctions can bypass the gate without exempting
  every version of a package.
- `pnpm outdated` applies the same maturity filter.
- If `latest` is too new, pnpm selects the highest mature version, even across
  a major-version boundary.

```yaml
minimumReleaseAgeExclude:
  - nx@21.6.5
  - "webpack@4.47.0 || 5.102.1"
```

## Trust policy

`trustPolicy: no-downgrade` rejects a release whose trust evidence is weaker
than earlier releases. `trustPolicyExclude` grants narrow version exceptions.
Failures remain fatal for optional dependencies. Prerelease evidence is
ignored while selecting a stable release.

```yaml
trustPolicy: no-downgrade
trustPolicyExclude:
  - chokidar@4.0.3
  - "webpack@4.47.0 || 5.102.1"
```

`trustPolicyIgnoreAfter` skips checks for releases older than the configured
age. `trustLockfile: true` skips release-age and no-downgrade rechecks for
entries loaded from an already-reviewed lockfile; its default is `false`.

Trusted-publisher evidence receives its highest rank only with provenance.
Staged-publish metadata containing an `approver` ranks above trusted publishers
and provenance in trust comparisons.

## Audit

pnpm 10 supports vulnerability filtering:

```sh
pnpm audit --ignore-unfixable
pnpm audit --ignore=CVE-2021-1234 --ignore=CVE-2021-5678
```

The first option omits vulnerabilities without a fix. Repeatable `--ignore`
omits selected CVEs even if a fix exists.

During pnpm 11 migration, replace `auditConfig.ignoreCves` with
`auditConfig.ignoreGhsas` and manually map each CVE to the GHSA in the audit
report. Newer pnpm introduces a top-level `audit` section:

```yaml
audit:
  level: high
  ignore:
    - GHSA-xxxx-yyyy-zzzz
```

It replaces `auditConfig` and `auditLevel`. Deprecated keys remain until the
next major, but the new values win when both exist.

`pnpm audit --fix=update` updates vulnerable packages in the lockfile rather
than writing overrides; `--interactive` selects advisories. Any audit fix adds
the minimum patched version to `minimumReleaseAgeExclude`, preventing the age
gate from delaying a security repair.

```sh
pnpm audit --fix=update --interactive
```

`pnpm audit signatures` validates installed packages against registry ECDSA
signatures and keys from `/-/npm/v1/keys`. It respects scoped registries and
skips registries that publish no signing keys.

## Native registry commands and authentication challenges

pnpm 11 implements `publish`, `view`, `login`, `logout`, `deprecate`,
`unpublish`, `dist-tag`, `version`, `search`, `star`, and `whoami` natively.
Several historical npm passthroughs instead report “not implemented,” including
`access`, `bugs`, `edit`, `issues`, `owner`, `prefix`, `profile`, `pkg`, `repo`,
`set-script`, `team`, `token`, and `xmas` in the initial pnpm 11 release.
`access` and `team` gained native implementations later in pnpm 11.

Publishing reads one-time passwords from `PNPM_CONFIG_OTP`, prompts when needed,
and supports browser authentication through a QR code and URL. Against
npmjs.org, `pnpm dist-tag add` and `rm` also surface a browser challenge when
`--otp` is omitted; `--otp=<code>` retains the classic flow.

Current native `pnpm access` manages visibility, collaborators, MFA, and team
grants. `pnpm team` supports `create`, `destroy`, `add`, `rm`, and `ls`, with
OTP, parseable, and JSON output:

```sh
pnpm team create @org:team --registry <url>
```

## Packing manifests

The `beforePacking` hook runs just before `pack` or `publish` creates the
tarball and returns the manifest to publish without changing local
`package.json`:

```js
module.exports = {
  hooks: {
    beforePacking(pkg) {
      delete pkg.devDependencies
      return pkg
    }
  }
}
```

Preview file inclusion without creating a tarball:

```sh
pnpm pack --dry-run
```

`pnpm pack --skip-manifest-obfuscation` and the matching publish option preserve
the original `packageManager` field and publish lifecycle scripts in the packed
manifest. The pnpm-specific `pnpm` field is still removed.

`pnpm publish` accepts a local `.tar.gz` package archive. A package may use
`publishConfig.engines` to publish runtime requirements different from its
development `engines`.

## Staged and atomic publishing

`pnpm stage` publishes a version hidden from ordinary installs, then permits
inspection, download, promotion, or rejection:

```sh
pnpm stage publish
pnpm stage list
pnpm stage view
pnpm stage download
pnpm stage approve
pnpm stage reject
```

Publish selected workspace packages atomically when the registry implements
pnpm's batch endpoint:

```sh
pnpm publish --recursive --batch
```

The registry receives one all-or-nothing request. An unsupported registry
causes `ERR_PNPM_BATCH_PUBLISH_UNSUPPORTED` rather than falling back to
non-atomic publishes.

## SBOM output

`pnpm sbom` emits CycloneDX or SPDX JSON. CycloneDX versions 1.5, 1.6, and 1.7
are selectable; 1.7 is the default, and the option is invalid for other output
formats.

```sh
pnpm sbom --sbom-format cyclonedx --sbom-spec-version 1.6
pnpm sbom --out sbom.json
pnpm sbom --split
pnpm sbom --exclude-peers
```

`--split` writes one file per selected workspace package; a single filtered
package becomes the root component. CycloneDX marks components reachable only
through development dependencies as excluded development components.
`--exclude-peers` removes peers and transitive subtrees reachable only through
those peers.

## Exotic transitive sources

`blockExoticSubdeps` rejects exotic transitive dependencies such as `git+ssh:`
repositories and direct HTTPS tarballs. Direct dependencies may still use
them. This protection defaults to enabled in pnpm 11.
