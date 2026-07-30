# Issuers and External Providers

Use this reference for Vault, Venafi, DNS providers, issuer readiness, and
provider-specific authentication or failure behavior.

## Vault

Vault issuers can set the TLS server name used to validate the Vault server's
certificate (since 1.18). Use it when the connection address and certificate
identity differ.

ServiceAccount tokens generated for Vault issuers include the Vault server
address in their default audiences from 1.20.

Vault issuers can authenticate with AWS IAM in 1.21 through IRSA, EKS Pod
Identity, or ambient EC2/ECS credentials rather than a long-lived AWS Secret.

Vault issuer validation rejects `..` path segments in `spec.vault.path` and in
authentication mount paths from 1.21. Do not depend on path joining to resolve
parent traversal.

The 1.21 chart stops creating token-request RBAC for the controller's own
ServiceAccount. If Vault Kubernetes authentication selects that account with
`serviceAccountRef.name`, add explicit Role and RoleBinding resources or use a
dedicated ServiceAccount with its own RBAC.

## Venafi

Username/password authentication can use a custom client ID from 1.17 rather
than the previous fixed default.

The `venafi.cert-manager.io/custom-fields` annotation added in 1.20 supplies
base custom fields on an Issuer or ClusterIssuer. Certificate-level custom
fields can override or append to that base.

From 1.21, invalid OAuth credentials produce the `AuthFailed` condition reason
so they can be distinguished from transient failures. PANW NGTS is also
supported as a Venafi backend.

## Azure DNS

When managed identities are used with service principals, AzureDNS accepts
`tenantID` from 1.17. Set it for explicit tenant selection in multi-tenant
environments.

Azure DNS-01 supports private zones through the `zoneType` field from 1.20:

```yaml
spec:
  acme:
    solvers:
      - dns01:
          azureDNS:
            zoneType: AzurePrivateZone
```

## Cloudflare, CloudDNS, and DigitalOcean

Use cert-manager 1.17.1 or later for Cloudflare DNS-01; that patch restored
issuance after a breaking Cloudflare API change.

From 1.20, the CloudDNS solver cleans up ACME challenge TXT records even when
the DNS name contains a large resource-record set.

DigitalOcean DNS-01 retries are regulated from 1.20. Complete DNS-01 failures
are attached to the Challenge as events, so inspect events before reducing a
failure to the final condition text.

## RFC2136

RFC2136 DNS-01 configuration has an explicit `protocol` field from 1.19. Use it
to select the DNS update transport, such as `TCP`, rather than depending on an
implicit choice.

## Issuer validation and response safety

DNS issuer credentials are validated before the issuer becomes Ready from
1.21. A Secret error should therefore appear as readiness failure instead of an
issuer that silently looks usable.

From 1.18.5, a certificate response whose public key does not match the CSR is
rejected before being stored. Issuance backs off instead of entering an
infinite loop.

From 1.21, an issuer response containing an already-expired certificate also
stops rather than entering an infinite reissuance loop.

## Issuer discovery

`Issuer` and `ClusterIssuer` have the kubectl short names `iss` and `ciss`
from 1.18:

```console
kubectl get iss
kubectl get ciss
```

From 1.20, `.spec.issuerRef.group`, `.spec.issuerRef.kind`, and
`.spec.issuerRef.name` are selectable CRD fields. This supports queries such as:

```console
kubectl get certificates --field-selector spec.issuerRef.name=example-issuer
```
