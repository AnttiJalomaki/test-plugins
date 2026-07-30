# Platforms, Identity, and Storage

## OpenShift compatibility

Consul 1.21.0 supports OpenShift Container Platform 4.16, 4.17, and 4.18.
Do not infer IPv6 support from this compatibility: IPv6 remains unsupported on
OpenShift for the address-family feature introduced later.

OpenShift 4.19 and later also require the gateway-resource migration described
in [Service mesh and gateways](service-mesh-and-gateways.md).

## Kubernetes Pod Security Admission

Consul can be deployed and configured with Kubernetes Pod Security Admission
controls applied per namespace (1.21.0). Pod Security Admission replaces
PodSecurityPolicy for enforcing minimum pod-security requirements.

Apply the desired admission level to every namespace participating in the
Consul deployment and account for those controls in chart or manifest changes.

## Google Cloud snapshot storage

The Enterprise snapshot-agent sidecar for Consul on Kubernetes can send
snapshots to Google Cloud Storage (1.21.0). Its supported target set includes:

- local storage
- Amazon S3
- Azure Blob Storage
- Google Cloud Storage

## Azure Managed Identity for snapshots

The Enterprise snapshot agent can authenticate to Azure Blob Storage with
Azure Managed Service Identity (1.22.0). Use it when the deployment should
avoid static Azure storage credentials in snapshot-agent configuration.

## OIDC PKCE and JWT client authentication

PKCE is enabled by default for Consul UI OIDC login in 1.22.0. OIDC providers
can also authenticate the client with a JWT assertion instead of a client
secret.

When changing an existing identity-provider integration, check both sides:

- The UI flow must accommodate the default PKCE behavior.
- The provider and Consul client configuration must agree on client-secret or
  JWT-assertion authentication.
