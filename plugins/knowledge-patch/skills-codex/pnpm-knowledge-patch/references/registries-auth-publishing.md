# Registries, authentication, globals, and publishing

## Selecting registries

Pass a scoped registry directly without the older `--config.` prefix
(`2025-02`):

```sh
pnpm --@scope:registry=https://scope.example.com/npm install
```

The `jsr:` protocol installs JSR packages with an optional range
(`2025-04`):

```sh
pnpm add jsr:@foo/bar
pnpm add jsr:@foo/bar@^0.1
```

A scoped JSR dependency is stored under its ordinary package name and converted
to an npm-compatible alias on publish. `@jsr` defaults to
`https://npm.jsr.io/` unless `@jsr:registry` is configured.

```json
{
  "dependencies": {
    "@foo/bar": "jsr:^0.1.2"
  }
}
```

pnpm 11 supports named registry dependency specifiers. `gh:` is built in for
GitHub Packages, while `namedRegistries` defines other names (`11.1-11.3`):

```yaml
namedRegistries:
  work: https://npm.work.example.com/
```

```sh
pnpm add work:@corp/lib@^2.0.0
```

Authentication still comes from the URL's normal `.npmrc` entry. Override the
built-in `gh` mapping for GitHub Enterprise Server.

## Login and scoped credentials

`pnpm login --scope <scope>` writes a normalized `@scope:registry` mapping and
the token to pnpm's auth file; a missing leading `@` is inserted
(`11.1-11.3`):

```sh
pnpm login --scope acme
```

Starting in pnpm 10.34, an unscoped `_authToken`, `_auth`,
`username`/`_password`, `tokenHelper`, or inline `cert`/`key` is pinned when
loaded to the registry declared in that same configuration source
(`10.34.0`). A later `registry=` override cannot redirect it. Each unscoped
setting warns with its source and pinned URL.

## TLS and token helpers

pnpm 10.25 accepts inline `cert`, `ca`, and `key` values scoped to a registry
URL in `.npmrc`; older versions only understand `certfile`, `cafile`, and
`keyfile` (`2025-12`):

```ini
//registry.example.com/:ca=-----BEGIN CERTIFICATE-----...
```

pnpm 10.27 rejects environment-variable references inside `tokenHelper` and
registry-scoped `<url>:tokenHelper` values. Provide a concrete helper command
from an allowed configuration source.

## File-free CI authentication

pnpm 11.6–11.9 accepts URL-scoped registry settings through environment names
beginning with `npm_config_//` or `pnpm_config_//` (`11.6-11.9`). These names
contain punctuation that ordinary shells reject in identifiers, so pass them
with `env` or a CI facility that supports arbitrary names:

```sh
env "pnpm_config_//registry.npmjs.org/:_authToken=$NPM_TOKEN" pnpm install
```

Trusted environment values override project/workspace `.npmrc`, but not CLI
options. A `pnpm_config_` value wins over the corresponding `npm_config_` value.

Authentication keys may add a package scope after the registry URL so multiple
scopes on one host use different tokens; an unscoped token is the fallback:

```ini
@org-a:registry=https://npm.pkg.github.com/
@org-b:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:@org-a:_authToken=ORG_A_TOKEN
//npm.pkg.github.com/:@org-b:_authToken=ORG_B_TOKEN
//npm.pkg.github.com/:_authToken=FALLBACK_TOKEN
```

## Structured global authentication

In pnpm 11.10–11.17, `_auth` can carry registry-wide (`@`) and scope-specific
credentials together with their destination URL (`11.10-11.17`). It is trusted
only from global `config.yaml` or `pnpm_config__auth`, never project files.
Global configuration also accepts `registries` and `namedRegistries`.

```sh
export pnpm_config__auth='{"https://registry.npmjs.org":{"@":{"authToken":"npm-token"},"@org":{"authToken":"org-token"}}}'
```

## Package archives and manifest transformation

`pnpm publish` accepts a `.tar.gz` archive (`2025-09`):

```sh
pnpm publish ./package.tar.gz
```

`publishConfig.engines` replaces top-level `engines` in the published manifest,
so development and published runtime requirements may differ (`2025-11`):

```json
{
  "engines": { "node": ">=24" },
  "publishConfig": {
    "engines": { "node": ">=20" }
  }
}
```

Use `hooks.beforePacking` to transform the outgoing manifest without modifying
local `package.json`; see [configuration-hooks.md](configuration-hooks.md).

Preview the packed file list without creating a tarball (`2025-12`):

```sh
pnpm pack --dry-run
```

`pnpm pack` and `pnpm publish` normally obfuscate package-manager metadata.
`--skip-manifest-obfuscation` preserves the original `packageManager` field and
publish lifecycle scripts; the pnpm-specific `pnpm` field is still removed
(`11.1-11.3`).

## OTP and browser authentication

pnpm 11 native publishing reads OTPs from `PNPM_CONFIG_OTP`, prompts when an OTP
is required, and can display a QR code and URL for web authentication
(`11.0.0`).

Against npmjs.org, `pnpm dist-tag add` and `pnpm dist-tag rm` surface a browser
OTP challenge when `--otp` is omitted. `--otp=<code>` keeps the classic flow
(`11.4-11.5`).

## Atomic workspace publishing

`pnpm publish --recursive --batch` publishes every selected workspace package
in one all-or-nothing request (`11.6-11.9`):

```sh
pnpm publish --recursive --batch
```

The registry must implement pnpm's batch-publish endpoint; otherwise the
command fails with `ERR_PNPM_BATCH_PUBLISH_UNSUPPORTED`.

## Staged publishing

`pnpm stage` publishes a version hidden from ordinary installs, then lets an
operator inspect, download, promote, or discard it (`11.1-11.3`):

```sh
pnpm stage publish
pnpm stage list
pnpm stage view
pnpm stage download
pnpm stage approve
pnpm stage reject
```

Staged-publish metadata also participates in trust ranking; see
[security-audit-sbom.md](security-audit-sbom.md).

## Global installation groups

pnpm 11 isolates each global installation group under
`{pnpmHomeDir}/global/v11/{hash}/`, and places binaries in `PNPM_HOME/bin`
(`11.0.0`). Space-separated arguments create separate groups, while
comma-separated package names share one group and are removed together
(`11.1-11.3`):

```sh
pnpm add -g foo bar
pnpm add -g foo,bar qar
```

## Native access and team administration

pnpm 11.10–11.17 implements `pnpm access` for package visibility,
collaborators, MFA requirements, and team grants. `pnpm team` supports
`create`, `destroy`, `add`, `rm`, and `ls`, including OTP, parseable, and JSON
output (`11.10-11.17`):

```sh
pnpm team create @org:team --registry <url>
```
