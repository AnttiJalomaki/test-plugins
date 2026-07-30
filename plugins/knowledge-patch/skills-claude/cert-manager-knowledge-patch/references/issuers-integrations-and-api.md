# Issuers, Integrations, and API Clients

Use this reference for Vault, Venafi, and Azure integration settings, issuer
selection, and typed API client migrations.

## Vault issuers

### TLS server-name validation

Since `1.18`, a Vault Issuer can specify the server name used to validate the
certificate presented by the Vault server. Configure it when the URL host and
certificate identity differ; do not disable validation to work around a name
mismatch.

### Token audiences

In `1.20`, service account tokens generated for Vault issuers include the Vault
server address among their default audiences. Ensure the Vault authentication
role accepts that audience when validating projected tokens.

### AWS IAM authentication

In `1.21`, Vault issuers can authenticate through IRSA, EKS Pod Identity, or
ambient EC2/ECS credentials. These modes avoid storing a long-lived AWS Secret;
select the mechanism matching the workload environment and constrain its AWS
and Vault roles.

### Reject parent traversal

In `1.21`, validation rejects `..` path segments in `spec.vault.path` and in
authentication mount paths. Migrate configurations that depended on path
joining to explicit canonical Vault paths before upgrading.

## Venafi issuers

### Custom client ID

Since `1.17`, Venafi username/password authentication can use a custom client
ID instead of the fixed default.

### Layered custom fields

In `1.20`, the `venafi.cert-manager.io/custom-fields` annotation on an Issuer
or ClusterIssuer supplies base custom fields. Values on a Certificate can
override or append to that base. Keep shared policy at issuer scope and only
place request-specific differences on the Certificate.

### Authentication conditions and backends

In `1.21`, Venafi issuers use the `AuthFailed` condition reason to distinguish
invalid OAuth credentials from transient failures. Alert or retry based on the
reason rather than treating every readiness failure alike. PANW NGTS is also
supported as a Venafi backend.

## Azure DNS

### Explicit tenant selection

Since `1.17`, the AzureDNS provider accepts `tenantID` when managed identities
are used with service principals. Set it for explicit tenant selection in
multi-tenant environments.

### Private zones

Since `1.20`, the Azure DNS-01 solver accepts `zoneType` for private zones:

```yaml
spec:
  acme:
    solvers:
      - dns01:
          azureDNS:
            zoneType: AzurePrivateZone
```

## Issuer references and queries

### Resource short names

Since `1.18`, `Issuer` and `ClusterIssuer` have the short names `iss` and
`ciss`:

```console
kubectl get iss
kubectl get ciss
```

### Field-selectable references

In `1.20`, CRDs expose `.spec.issuerRef.group`, `.spec.issuerRef.kind`, and
`.spec.issuerRef.name` as selectable fields. Resources can be queried without a
client-side scan:

```console
kubectl get certificates --field-selector spec.issuerRef.name=example-issuer
```

### Avoid the 1.19.0 persisted defaults

Version 1.19.0 added CRD-level defaults for the issuer-reference group and kind
on Certificate and CertificateRequest. Those persisted defaults could trigger
unnecessary certificate reissuance and were reverted in 1.19.1. Use 1.19.1 or
later so omitted fields retain runtime defaulting behavior.

## Kubernetes API client changes

### Type-safe server-side apply

Since `1.19`, generated apply-configuration types are available for
cert-manager resources. Clients can construct type-safe server-side apply
requests rather than using unstructured apply payloads.

### Removed `ObjectReference`

In `1.21`, the deprecated `ObjectReference` API type was removed. Update API
clients and integrations to the applicable typed reference before upgrading.
