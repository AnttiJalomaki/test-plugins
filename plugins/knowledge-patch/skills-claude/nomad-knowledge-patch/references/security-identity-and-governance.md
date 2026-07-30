# Security, Identity, and Governance

Use this reference for ACL semantics, client and workload identity, OIDC, job
secrets, allocation authentication, and policy enforcement.

## ACL API and policy writes

In Nomad 1.10.1, `/v1/acl/token/self` returns status codes that distinguish
configuration from authentication failure:

- With ACLs disabled, it returns `200` and a body indicating that ACLs are
  disabled.
- With ACLs enabled but no valid token, it returns `403`.

Both cases previously returned `404`; update clients that treated `404` as
either condition (source batch `1.10-upgrade`).

Starting in 1.10.6, policy writes containing duplicate or invalid keys are
rejected rather than silently ignoring the bad keys. Existing affected
policies keep operating, but their source documents must be corrected before
they can be written again.

Workload identity tokens can list and retrieve policies through the ACL API
(source batch `1.11.0`). Review policy access independently from the workload's
other capabilities.

## OIDC client authentication

OIDC auth methods can use a private-key JWT client assertion instead of sending
a client secret. PKCE works with either client secrets or assertions. Enable it
with:

```text
OIDCEnablePKCE: true
```

The OIDC provider must support PKCE and may require it to be enabled in provider
configuration. Keep assertion-key handling, provider capabilities, and PKCE
configuration aligned (source batch `1.10.0`).

## Client introduction and RPC identity

Clients can join with signed JWT introduction tokens. Token constraints can
cover node names, node pools, and TTLs; server enforcement levels determine the
introduction policy and produce violation logs and metrics. After registration,
servers automatically issue and rotate a client identity for RPC
authentication. This identity is a second layer alongside mTLS.

```shell
nomad node intro create
nomad agent -client-intro-token <token>
nomad node identity get
nomad node identity renew
```

Use introduction tokens only for initial admission; use the issued identity
commands to inspect or renew post-registration credentials.

## Job specification secrets

The jobspec `secret` block fetches values from Nomad, Vault, or a custom
secret-provider plugin for interpolation. Reference a fetched key as:

```hcl
${secret.secret_name.key}
```

Keep provider selection, access policy, and interpolation scope explicit.
Secret-provider plugin execution has a 60-second timeout; slow operations now
fail at that boundary (source batch `2.0.0`).

## Consul and Vault allocation authentication

The deprecated token-based allocation authentication workflows for both
Consul and Vault have been removed. Migrate jobs and cluster configuration to
supported identity-based workflows.

A task with a `template` block no longer receives a Consul identity as a side
effect. Declare the identity the task actually needs instead of relying on the
presence of a template block (source batch `1.10.0`).

## Sentinel and volume governance

`nomad sentinel apply` requires the `-scope` option. Update scripts and operator
runbooks so every applied policy selects its intended scope explicitly.

Nomad Enterprise can evaluate dynamic host-volume specifications with Sentinel
during creation, enforce per-namespace host-volume capacity quotas, and
validate a requested node pool against the namespace's node-pool
configuration. Treat all three checks as possible causes of a rejected volume,
even when its storage-specific fields are valid.
