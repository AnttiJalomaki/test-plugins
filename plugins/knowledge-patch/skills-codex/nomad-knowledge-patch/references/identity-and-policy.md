# Identity and Policy

## Client introduction and RPC identity

Clients can join with signed JWT introduction tokens. Token constraints can
cover node names, node pools, and TTLs. Server enforcement levels control the
introduction policy and produce violation logs and metrics.

Create a token and pass it to the joining client:

```shell
nomad node intro create
nomad agent -client-intro-token=<token>
```

After registration, servers automatically issue and rotate a client identity
for RPC authentication. This identity is a second layer alongside mTLS.
Inspect or renew it with:

```shell
nomad node identity get
nomad node identity renew
```

Client introduction and identity are from batch `1.11.0`.

## Jobspec secrets

The `secret` block fetches secrets from Nomad, Vault, or a custom
secret-provider plugin for jobspec interpolation. Refer to a fetched key as:

```hcl
${secret.secret_name.key}
```

The block is from batch `1.11.0`. Secrets-plugin execution has a 60-second
timeout as of batch `2.0.0`; take that boundary into account for slow custom
providers.

## OIDC assertions and PKCE

OIDC auth methods support private-key JWT client assertions instead of a client
secret. PKCE works with either client secrets or assertions:

```text
OIDCEnablePKCE: true
```

The OIDC provider must support PKCE and may require it to be enabled separately.
These auth-method changes are from batch `1.10.0`.

## Allocation authentication migration

The deprecated token-based allocation authentication workflows for both Consul
and Vault were removed. Migrate jobs to the supported identity mechanisms. A
task containing a `template` block no longer receives a Consul identity as a
side effect, so never rely on the block for implicit authentication.

These removals are from batch `1.10.0`.

## ACL endpoint and policy behavior

`/v1/acl/token/self` uses these statuses as of 1.10.1:

- ACLs disabled: `200`, with a body indicating that ACLs are disabled.
- ACLs enabled but no valid token supplied: `403`.

Both cases previously returned `404`.

As of 1.10.6, policy writes containing duplicate or invalid keys are rejected
instead of silently ignoring the bad keys. Existing affected policies continue
to work, but their source documents must be fixed before they can be written
again. These upgrade behaviors are from batch `1.10-upgrade`.

Workload identity tokens can list or retrieve policies through the ACL API
(batch `1.11.0`).

## Sentinel and quota policy

`nomad sentinel apply` requires an explicit `-scope` option.

Enterprise dynamic host volume creation can:

- evaluate volume specifications with Sentinel;
- enforce per-namespace host-volume capacity quotas;
- validate a requested node pool against the namespace's node-pool
  configuration.

The command and volume-governance rules are from batch `1.10.0`.

## Quota API migration

The quota `variables_limit` field and Go API `QuotaSpec.VariablesLimit` are
deprecated for removal in 1.12. Replace them with:

```text
region_limit.storage.variables
QuotaSpec.RegionLimit.Storage.Variables
```

The Go API type of `QuotaSpec.RegionLimit` changes from `Resources` to
`QuotaResources`. These schema and API changes are from batch `1.10.0`.
