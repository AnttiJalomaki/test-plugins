# State, backends, locking, and installation

Use this reference for persistent state, backend authentication, lock
coordination, plugin caches, and package verification.

## S3 credentials

In 1.7.0, the S3 backend changed the default of `use_legacy_workflow` to
`false`. Credential resolution follows AWS CLI/SDK precedence and prefers
backend configuration over environment variables. The option was a temporary
compatibility escape hatch:

```hcl
terraform {
  backend "s3" {
    use_legacy_workflow = true
  }
}
```

OpenTofu 1.8.0 removes the argument; remove it before upgrading. Standard
credential discovery is the only supported behavior.

S3 module source addresses adopt AWS CLI/SDK-style credential discovery in
1.11.0. The selected source can therefore change after upgrade, and schemes
such as IAM roles for service accounts become available.

The S3 backend recognizes credentials issued by `aws login` in 1.12.0.

## S3 locking and object behavior

The 1.10 S3 backend can lock state in S3 itself without DynamoDB. Use 1.10.2 or
later when the bucket requires server-side encryption: that patch sends the
`x-amz-server-side-encryption` header for the lockfile.

The S3 backend can tag state-snapshot and lock objects in 1.11 and can use the
`eusc-de-east-1` AWS European Sovereign Cloud region.

`skip_s3_checksum` in 1.10 also disables the AWS SDK's S3 integrity checks.
That may help incomplete S3-compatible implementations, but it broadens what
the setting bypasses.

## PostgreSQL layout and locking

The 1.10 `pg` backend accepts `table_name` and `index_name`, allowing separate
tables for multiple states in one database. Finer-grained locks stop unrelated
configurations from contending.

Do not run 1.10 and older OpenTofu processes against the same database. Their
incompatible locking implementations can permit conflicting writes and data
loss.

## AzureRM backend

The backend accepts `timeout_seconds` since 1.9, with a 300-second default.

In 1.11, it ignores deprecated `endpoint`/`ARM_ENDPOINT` and
`msi_endpoint`/`ARM_MSI_ENDPOINT`. Use `MSI_ENDPOINT` instead of the latter,
and do not set `environment` together with `metadata_host`. Refresh an existing
working directory with:

```bash
tofu init -reconfigure
```

Use `-reconfigure`, not `-migrate-state`, because the change does not move
state.

New 1.11 authentication settings include:

- `use_cli`, defaulting to `true`.
- `use_aks_workload_identity`, defaulting to `false`.
- `client_id_file_path` and `client_secret_file_path`.
- Inline `client_certificate`.

In 1.12, AzureRM adds Azure DevOps and Azure Pipelines workload identity
federation. It also supports Customer-Provided Keys and Customer-Managed Keys
for backend server-side encryption.

## HTTP, OSS, and GCS backends

The HTTP backend accepts custom request headers since 1.7. Its trace logs
include request and response bodies since 1.9, so protect trace output from
credential or state disclosure.

The HTTP backend supports `tofu force-unlock` since 1.10. The OSS backend
honors standard proxy variables, including `NO_PROXY`.

The GCS backend adds `universe_domain` in 1.11.5 for sovereign GCP services.

## Local persistence

Since 1.7, the local state manager no longer persists every in-memory
`state.Write()` immediately. A hard crash during apply may leave no
in-progress state file to inspect, matching other state managers.

`TF_STATE_PERSIST_INTERVAL` controls the state-write interval since 1.8.

## Concurrent provider caches

Since 1.10, multiple OpenTofu processes can share the global provider cache
when its filesystem supports file locking. Use 1.10.5 or later to avoid a
lock-contention bug with `TF_PLUGIN_CACHE_DIR`. Every project using the cache
must retain a valid `.terraform.lock.hcl`.

## Provider source compatibility and checksums

During `tofu init` in 1.10, a lock entry for certain providers on
`registry.terraform.io` may select the same version republished on
`registry.opentofu.org`. This applies only to providers OpenTofu rebuilds and
republishes, not arbitrary third-party providers.

For unsigned provider ZIP sources, the lock file records a locally verified
`zh:` archive checksum alongside `h1:`. This improves verification when the
same provider is installed again from another source.

When 1.12 installs directly from OpenTofu Registry, `tofu init` records the
complete set of `zh:` and `h1:` hashes for every supported platform. The first
initialization after upgrade may add many `h1:` entries to
`.terraform.lock.hcl`.

`tofu providers lock` remains necessary when installation uses an alternative
source. A `network_mirror` may opt to trust all hashes reported by that mirror.

## Module and provider distribution

Since 1.10, module packages accept the `oci:` source-address scheme and OCI
registries can serve as provider mirrors. These surfaces support registry-based
distribution in connected and air-gapped environments.

A module registry may tell 1.12 clients to reuse registry API credentials for
package downloads. This removes the need for a separate `.netrc` credential
when the registry itself serves the package.

For a non-AWS origin, an S3 module source beginning with `s3::http://` uses
plaintext HTTP in 1.12 instead of silently changing to HTTPS. Official AWS
hostnames remain the exception; review non-AWS URLs before relying on the new
behavior.
