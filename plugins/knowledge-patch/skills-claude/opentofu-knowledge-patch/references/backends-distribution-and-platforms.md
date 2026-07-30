# Backends, distribution, and platforms

## S3 credentials, locking, and object behavior

OpenTofu 1.7.0 changes the S3 backend default for `use_legacy_workflow` to
`false`. Credential lookup follows AWS CLI and SDK precedence and prefers
backend configuration over environment variables. Setting the deprecated
option to `true` provides only temporary compatibility.

OpenTofu 1.8.0 removes `use_legacy_workflow`; remove it from configuration.

OpenTofu 1.10.0 adds native S3 state locking without DynamoDB. Use 1.10.2+
when the bucket requires server-side encryption so the lockfile request sends
`x-amz-server-side-encryption`.

Other S3 backend changes include:

- In 1.10, `skip_s3_checksum` also disables the AWS SDK's S3 integrity checks.
  This may help incomplete S3-compatible implementations, but broadens what
  the setting bypasses.
- In 1.11, the backend can tag state-snapshot and lock objects and use buckets
  in the `eusc-de-east-1` AWS European Sovereign Cloud region.
- In 1.12.0, it discovers credentials issued by `aws login`.

## S3 module downloads

S3 module source addresses use AWS CLI/SDK credential discovery in 1.11.0,
replacing OpenTofu's custom sequence. This can select different credentials
after upgrade and supports mechanisms such as IAM roles for service accounts.

Since 1.12.0, `s3::http://` preserves plaintext HTTP for a non-AWS origin
instead of silently upgrading to HTTPS. Official AWS hostnames remain the
exception.

## HTTP, OSS, GCS, and AzureRM backends

The HTTP backend accepts user-defined request headers since 1.7.0. In 1.9.0,
HTTP backend trace logs include request and response bodies; handle logs as
potentially sensitive. In 1.10, HTTP supports `tofu force-unlock`.

The OSS backend honors standard proxy variables, including `NO_PROXY`, from
1.10.0.

The AzureRM backend adds `timeout_seconds` with a default of 300 seconds in
1.9.0. OpenTofu 1.11.0 ignores deprecated `endpoint`/`ARM_ENDPOINT` and
`msi_endpoint`/`ARM_MSI_ENDPOINT`; use `MSI_ENDPOINT`, and do not combine
`environment` with `metadata_host`. Refresh the working directory with:

```bash
tofu init -reconfigure
```

Do not use `-migrate-state` for this authentication transition because it does
not relocate state.

AzureRM 1.11 authentication controls include:

- `use_cli`, defaulting to `true`
- `use_aks_workload_identity`, defaulting to `false`
- `client_id_file_path`
- `client_secret_file_path`
- inline `client_certificate`

OpenTofu 1.12.0 adds Azure DevOps and Azure Pipelines workload identity
federation plus Customer-Provided Keys and Customer-Managed Keys for
server-side encryption.

The GCS backend adds `universe_domain` in 1.11.5 for sovereign GCP services.

## PostgreSQL layout and locking

OpenTofu 1.10.0 adds `table_name` and `index_name` to the `pg` backend, so
separate state collections can use separate tables in one database.
Finer-grained locking prevents unrelated configurations from contending.

Do not mix 1.10 and older OpenTofu processes in the same database. Their
locking schemes are incompatible and can permit conflicting writes and data
loss.

## State persistence and local crashes

Since 1.7.0, the local state manager no longer persists every in-memory
`state.Write()` immediately. A hard crash during apply can leave no
in-progress state file to inspect, matching other state managers.

OpenTofu 1.8.0 adds `TF_STATE_PERSIST_INTERVAL` to control the state-write
interval.

## OCI modules and provider mirrors

OpenTofu 1.10.0 adds the `oci:` module source scheme and lets OCI registries
serve as provider mirrors. These support registry-based distribution for both
artifact types, including air-gapped environments.

## Global cache concurrency and lock files

Multiple 1.10 OpenTofu processes can share the global provider cache when its
filesystem supports file locking. Use 1.10.5+ to avoid a
`TF_PLUGIN_CACHE_DIR` lock-contention bug, and maintain a valid
`.terraform.lock.hcl` in every project using the shared cache.

When `tofu init` installs directly from OpenTofu Registry, 1.12.0 records all
supported-platform `zh:` and `h1:` hashes. The first initialization after
upgrade can therefore add many `h1:` entries. Continue using
`tofu providers lock` with alternative installation sources. A
`network_mirror` may opt to trust all hashes reported by that mirror.

Since 1.10, an eligible lock entry for a provider on
`registry.terraform.io` can select the same version rebuilt and republished
on `registry.opentofu.org`. This applies only to providers OpenTofu rebuilds,
not arbitrary third-party providers.

For unsigned provider ZIP sources, OpenTofu records a locally verified `zh:`
archive checksum alongside `h1:`, improving verification when reinstalling
from a different source.

## Private registry credentials

OpenTofu 1.12.0 lets a module registry instruct OpenTofu to reuse its API
credentials for package downloads. This avoids a separate `.netrc` credential
when the registry itself serves the package.

Registry retry counts and request timeouts are configurable in CLI
configuration as well as environment variables since 1.11.0.

## Platform and transport boundaries

OpenTofu 1.6 is unsupported as of 1.9.0 and receives no further security
updates; move to at least 1.7.

OpenTofu 1.10.0 requires Linux kernel 3.2+ or macOS 11+. On Windows, junctions
are no longer treated as symlinks; a `TEMP` path traversing a junction can
fail and should use a real directory symlink.

OpenTofu 1.11.0 requires macOS 12 Monterey or later, rejects SHA-1 signatures
during TLS handshakes, and rejects malformed SSH certificates whose signing
key is itself a certificate key. Use 1.11.4+ for installations that may
process untrusted provider or module ZIP files; earlier 1.11 releases can
spend excessive time on a malicious archive.

OpenTofu 1.12.0 is the last planned macOS 12 series. WinRM still works but
warns and is planned to become an error in 1.13, so migrate Windows
provisioners to SSH. Official 32-bit `386` and `arm` packages continue through
1.13 but are planned for later removal; `amd64` and `arm64` are unaffected.

Early 1.12 releases had security defects in SSH connections, OpenBao-wrapped
encryption data, revoked SSH CA handling, and malicious Git URLs that could
read arbitrary files. Use 1.12.4 or later.

## Container-image boundary

Using `ghcr.io/opentofu/opentofu` as a base for custom images is deprecated in
1.9.0 and unsupported in 1.10.0. Use a supported image-building strategy
before upgrading.

## XDG paths and login browser

OpenTofu supports XDG Base Directory locations for its files since 1.7.0.

On Unix, `tofu login` in 1.12.0 honors `BROWSER` only when it names a single
command that accepts the URL as its sole argument. An existing environment
value can therefore change browser-launch behavior.
