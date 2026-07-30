# Kubernetes, OpenShift, and Snapshots

## Apply Pod Security Admission

Consul can be deployed with Kubernetes Pod Security Admission controls applied
per namespace (since 1.21.0). Use Pod Security Admission, rather than the
PodSecurityPolicy mechanism it replaces, to enforce minimum pod security
requirements. Align namespace policy levels with the Consul workloads placed in
each namespace.

## Match Supported OpenShift Releases

Consul 1.21.0 supports OpenShift Container Platform 4.16, 4.17, and 4.18.
Check the Consul and OpenShift pairing explicitly before an orchestrator
upgrade.

IPv6 behavior added with Consul 1.22.0 does not extend to OpenShift in that
release. Do not assume VM or Kubernetes IPv6 guidance applies unchanged to an
OpenShift deployment.

## Migrate Gateway Resources on OpenShift

OpenShift 4.19 and later require the newer Kubernetes resource types in the
`consul.hashicorp.com` API group (since 2.0.0). Earlier Kubernetes Gateway API
`v1alpha` resources are incompatible. Migrate existing gateway resources as
part of the OpenShift upgrade rather than waiting for gateway reconciliation
to fail.

## Scale Enterprise API Gateways

Enterprise API Gateways on Kubernetes can scale beyond the earlier
eight-replica ceiling (since 2.0.0). Horizontal Pod Autoscaling is supported
when enabled through annotations on the Gateway resource. Coordinate replica
limits, annotations, and cluster capacity rather than carrying forward an
eight-replica assumption.

## Store Snapshots in Google Cloud Storage

The Enterprise snapshot-agent sidecar for Consul on Kubernetes can send
snapshots to Google Cloud Storage (since 1.21.0). This joins the supported
local, Amazon S3, and Azure Blob Storage targets. Select the destination and
credentials explicitly for the deployed sidecar.

## Authenticate Azure Snapshots with Managed Identity

The Enterprise snapshot agent can authenticate to Azure Blob Storage with
Azure Managed Service Identity (since 1.22.0). Prefer managed identity when
the environment supports it and static storage credentials should not be
stored in snapshot-agent configuration.
