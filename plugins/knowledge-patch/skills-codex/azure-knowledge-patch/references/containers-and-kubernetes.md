# Containers And Kubernetes

Use this reference for AKS, node pools, networking, safeguards, Azure Container Storage, ACR, and Container Apps.

## ACR access and source-registry identity

_Azure CLI `2.73.0`._

Registry create/update accepts `--role-assignment-mode` to enable or disable
ABAC, and `az acr check-health --repository` checks read, write, and delete
permissions for one repository. ACR task create/update, build, and run accept
`--source-acr-auth-id` to choose the managed identity used to authenticate to
the source registry.

## ACR content-trust and health-check breaking changes

_Azure CLI `2.83.0`._

`az acr config content-trust update` announces that the `enabled` status will
stop being accepted. `az acr check-health` also announces removal of its
Notary client check, so automation must not depend on either behavior.

## ACR content-trust command deprecation

_Azure CLI `2.79.0`._

`az acr config content-trust` and its `show` and `update` operations now emit
deprecation notices. Treat these interfaces as transitional in automation.

## ACR creation controls

_Azure CLI `2.88.0`._

`az acr create` now accepts `--data-endpoint-enabled` for a dedicated data
endpoint used in client firewall configuration and `--endpoint-protocol` to
select the registry endpoint protocol during creation.

## ACR domain-name-label hash scope

_Azure CLI `2.72.0`._

`az acr create` and `az acr check-name` accept `--dnl-scope` to select the
scope used for the registry's domain-name-label hash.

## ACR endpoint routing and cache identities

_Azure CLI `2.85.0`._

`az acr replication create` and `update` gain
`--global-endpoint-routing`; `--region-endpoint-enabled` now redirects to
that option rather than being confused with registry-level
`--regional-endpoints`. Cache-rule create/update also accepts `--identity`
for a user-assigned managed identity.

## ACR exposed-token output

_Azure CLI `2.74.0`._

`az acr login --expose-token` now adds `refreshToken` and `username` fields to
its output, so consumers with fixed output schemas must account for them.

## ACR Podman and regional-endpoint login

_Azure CLI `2.86.0`._

`az acr login --name` now supports Podman. For registries with regional
endpoints enabled, `az acr show-endpoints` displays their host names,
`az acr login --endpoint` selects one for login, and `az acr import` accepts a
regional endpoint URI as its source.

## ACR token audience

_Azure CLI `2.82.0`._

`az acr login` now enforces the ACR audience when acquiring its Microsoft
Entra token. Token acquisition policies or scripts that assumed another
audience must account for this change.

## ACR token audience and endpoint protocol

_Azure CLI `2.87.0`._

`az acr login` can now customize the Microsoft Entra token audience used for
authentication. Registry updates also accept `--endpoint-protocol` to select
the registry endpoint protocol.

## AKS advanced networking controls

_Azure CLI `2.68.0`._

`az aks create` and `az aks update` gain `--enable-acns`; when enabling it,
`--disable-acns-observability` and `--disable-acns-security` can omit the
corresponding ACNS features. `az aks update` also gains `--disable-acns`.

```bash
az aks update --resource-group "$RESOURCE_GROUP" --name "$CLUSTER" \
  --enable-acns --disable-acns-observability
```

## AKS application auto-instrumentation

_Azure CLI `2.86.0`._

`az aks update --enable-azure-monitor-app-monitoring` enables Azure Monitor
Application Monitoring auto-instrumentation on a cluster.

## AKS artifact streaming

_Azure CLI `2.87.0`._

AKS add and update operations accept `--enable-artifact-streaming` and
`--disable-artifact-streaming`, allowing artifact streaming to be toggled
through the CLI.

## AKS attach-ACR principal types

_Azure CLI `2.75.0`._

When `az aks update` uses `--attach-acr`, the new
`--assignee-principal-type` option specifies the attached registry assignee's
principal type.

## AKS Automatic clusters on a bring-your-own VNet

_Azure CLI `2.86.0`._

`az aks create` can combine `--enable-hosted-system`,
`--system-node-subnet-id`, and `--node-subnet-id` to place an Automatic
Managed System Pool cluster on a caller-supplied virtual network.

## AKS Azure Container Storage v2 controls

_Azure CLI `2.83.0`._

On a new cluster, `az aks create --enable-azure-container-storage` enables
ACStor v2 without selecting a storage option. On an existing cluster,
`az aks update --enable-azure-container-storage ephemeralDisk` enables
ephemeral-disk storage, `--disable-azure-container-storage elasticSan`
disables Elastic SAN storage, and the value-less disable flag disables ACStor
v2 entirely.

## AKS bootstrap artifacts and outbound type

_Azure CLI `2.71.0`._

`az aks create` and `az aks update` accept `--bootstrap-artifact-source` and
`--bootstrap-container-registry-resource-id` for selecting the cluster's
bootstrap artifact source and registry. Their `--outbound-type` option also
accepts `none`.

```bash
az aks update --resource-group "$RESOURCE_GROUP" --name "$CLUSTER" \
  --bootstrap-artifact-source "$ARTIFACT_SOURCE" \
  --bootstrap-container-registry-resource-id "$REGISTRY_ID" \
  --outbound-type none
```

## AKS container-network logs and cloud workspaces

_Azure CLI `2.84.0`._

`az aks create` gains `--enable-container-network-logs`; `az aks update` can
toggle the feature with `--enable-container-network-logs` and
`--disable-container-network-logs`. `az aks enable-addons` can now create a
default workspace in the Bleu and Delos clouds.

## AKS control-plane metrics

_Azure CLI `2.88.0`._

AKS create can enable Azure Monitor managed Prometheus control-plane metrics
with `--enable-control-plane-metrics` or `--enable-cp-metrics`; update can
also disable them with `--disable-control-plane-metrics` or
`--disable-cp-metrics`.

## AKS custom-CA removal and advanced network policies

_Azure CLI `2.79.0`._

`az aks update` can remove existing custom CA certificates by passing an empty
file to `--custom-ca-trust-certificates`. AKS create and update also accept
`--acns-advanced-networkpolicies` with `None`, `L7`, or `FQDN`.

## AKS deployment safeguards and run-command policy

_Azure CLI `2.76.0`._

The new `az aks safeguards` group manages deployment safeguards. Creation can
disable run command with `--disable-run-command`, while update can toggle it
with `--disable-run-command` or `--enable-run-command`.

## AKS device-code kubeconfigs

_Azure CLI `2.78.0`._

`az aks get-credentials` converts device-code-mode kubeconfigs to Azure CLI
token format so that conditional-access login does not block them.

## AKS downloads and node-pool choices

_Azure CLI `2.82.0`._

`az aks install-cli` accepts `--gh-token` to authenticate the GitHub download
of `kubelogin`. `az aks nodepool add` and `update` accept `Ubuntu2404` for
`--os-sku`, and node-pool update now accepts `--gpu-driver install` or
`--gpu-driver none`.

## AKS isolation and node networking

_Azure CLI `2.80.0`._

Cluster and node-pool creation accept `KataVmIsolation` for
`--workload-runtime`; node-pool add and update accept `--localdns-config`.
Service Mesh egress gateways can be managed with
`az aks mesh enable-egress-gateway` and
`az aks mesh disable-egress-gateway`.

## AKS Istio CNI and managed gateways

_Azure CLI `2.86.0`._

`az aks mesh enable` and `az aks mesh proxy-redirection-mechanism` add Istio
CNI management. AKS create/update can toggle Managed Gateway API with
`--enable-gateway-api` or `--disable-gateway-api`, and the App Routing Istio
gateway implementation with `--enable-app-routing-istio` or
`--disable-app-routing-istio`.

## AKS load-balancer SKU migration

_Azure CLI `2.76.0`._

`az aks update` can now migrate a cluster load balancer from Basic to Standard
SKU.

## AKS maintenance-window format

_Azure CLI `2.88.0`._

`az aks maintenanceconfiguration add` and `update` now accept the
`maintenanceWindow` format for the default maintenance configuration.

## AKS managed namespaces

_Azure CLI `2.80.0`._

The new `az aks namespace` group supports `add`, `update`, `show`, `list`,
`delete`, and `get-credentials` operations for managed namespaces.

## AKS message of the day

_Azure CLI `2.70.0`._

`az aks create` and `az aks nodepool add` accept `--message-of-the-day`, so a
message can be configured with either the cluster or a newly added node pool.

## AKS network-family updates and node-resource-group lockdown

_Azure CLI `2.68.0`._

`az aks update` can now change the cluster network with `--ip-families`.
Both create and update accept `--nrg-lockdown-restriction-level` to set the
managed node resource group's restriction level.

## AKS network-integration behavior

_Azure CLI `2.73.0`._

AKS create/update now supports API-server VNet integration. Cluster creation
and app routing also apply a default NIC configuration for app routing.

## AKS networking and high-volume logging

_Azure CLI `2.85.0`._

AKS create/update accepts `--acns-transit-encryption-type` with `WireGuard`
or `None` for pod-to-pod encryption and adds ACNS performance support.
`az aks update` also gains `--enable-http-proxy`, `--disable-http-proxy`, and
`--enable-high-log-scale-mode` for proxy and Container Logs configuration.

## AKS node operating-system choices

_Azure CLI `2.86.0`._

`AzureContainerLinux` is now accepted by `az aks create` and by node-pool
`add` and `update` through `--os-sku`. Node-pool `add` also accepts
`Windows2025`.

## AKS node OS and container-storage validation

_Azure CLI `2.78.0`._

`az aks nodepool add` and `update` accept `AzureLinux3` for `--os-sku`.
Creating a cluster with v1 container storage now fails when the VM SKU is empty.

## AKS node VM-size default

_Azure CLI `2.73.0`._

The `--node-vm-size` default is now an empty string for `az aks create` and
`az aks nodepool add`; pass a value explicitly when provisioning must use a
specific VM size.

## AKS node-pool disruption and soak controls

_Azure CLI `2.70.0`._

`az aks nodepool delete` accepts `--ignore-pod-disruption-budget` when a
deletion must proceed despite PodDisruptionBudgets. An upgrade can now set
`--node-soak-duration` to `0` when no soak interval is wanted.

## AKS node-pool OS, CA trust, and GPU driver controls

_Azure CLI `2.72.0`._

`az aks nodepool add` and `az aks nodepool update` accept `Ubuntu2204` for
`--os-sku`. Cluster creation and node-pool addition gain
`--custom-ca-trust-certificates`, while node-pool addition can explicitly use
`--gpu-driver install` or `--gpu-driver none`.

## AKS node-pool upgrade and rollback

_Azure CLI `2.88.0`._

`az aks nodepool upgrade` no longer silently ignores `--max-unavailable`.
The new `az aks nodepool get-rollback-versions` and `rollback` commands list
rollback versions and restore an agent pool to its most recently used
configuration.

## AKS provisioning and observability add-ons

_Azure CLI `2.76.0`._

AKS create/update gains `--node-provisioning-mode`,
`--node-provisioning-default-pools`, and `--enable-ai-toolchain-operator` for
the Kaito add-on. Cluster creation can also configure the Azure Monitor
metrics and logs add-on.

## AKS SSH-key default

_Azure CLI `2.80.0`._

`az aks create` now uses `--no-ssh-key` behavior by default, enacting the
breaking change announced in 2.78.0.

## AKS SSH-key default warning

_Azure CLI `2.78.0`._

This release pre-announces a breaking change to the default behavior of
`az aks create --no-ssh-key`; automation should not rely on its implicit default.

## AKS static egress gateways

_Azure CLI `2.75.0`._

`az aks create` and `az aks update` accept
`--enable-static-egress-gateway`. To add the corresponding gateway node pool,
`az aks nodepool add` accepts `Gateway` for `--mode` together with
`--gateway-prefix-size`.

## AKS upgrade availability controls

_Azure CLI `2.74.0`._

`az aks nodepool add`, `update`, and `upgrade` accept
`--undrainable-node-behavior` to control whether nodes can be cordoned during
an upgrade and `--max-unavailable` to cap simultaneously unavailable nodes by
number or percentage. The preview designation is also removed from
`--enable-high-log-scale-mode` on `az aks create` and `enable-addons`.

## AKS virtual-machine pools and migration

_Azure CLI `2.76.0`._

AKS commands now support Virtual Machines node pools, and `az aks update` can
migrate an agent pool from VMAS to VMS. `az aks machine show/list` also adds
zones to table output.

## App Configuration import from AKS ConfigMaps

_Azure CLI `2.77.0`._

`az appconfig kv import` can now import key-values from an AKS ConfigMap.

## Azure Container Storage lifecycle and AKS Automatic

_Azure CLI `2.77.0`._

`az aks create` and `az aks update` can install the latest acstor release with
`--enable-azure-container-storage`; when enabling it,
`--container-storage-version` selects a specific release. `az aks update
--disable-azure-container-storage` can uninstall acstor regardless of its
installed version, and create/update also accept `--sku` for AKS Automatic.

## Connected-registry garbage collection

_Azure CLI `2.73.0`._

`az acr connected-registry create` and `update` accept `--gc-enabled` and
cron-based `--gc-schedule` to control garbage collection.

## Container App Compose environment parsing

_Azure CLI `2.69.0`._

`az containerapp compose create` splits an environment assignment only at
its first `=`, so values can themselves contain equal signs.

## Container App job listing and zero execution limits

_Azure CLI `2.77.0`._

`az containerapp job list` no longer stops after 20 items. `az containerapp
job update` now accepts `0` for both `--min-executions` and
`--max-executions`.

## Container Apps environment routing and premium ingress

_Azure CLI `2.79.0`._

The new `az containerapp env http-route-config` and
`az containerapp env premium-ingress` groups manage environment-level HTTP
routing and premium ingress settings.

## Container Apps infrastructure resource groups

_Azure CLI `2.82.0`._

`az containerapp env create --infrastructure-resource-group` selects the
resource-group name used for the environment's infrastructure resources.

## Container-group defaults removed

_Azure CLI `2.76.0`._

`az container create` no longer injects its former container-group defaults,
allowing standby-pool reuse. Automation that depended on those CLI defaults
must now pass the required values explicitly.

## Credentialless ACR cache rules

_Azure CLI `2.71.0`._

`az acr create` can now create a cache rule without a credential set; that
previously failed even when the cache rule did not need credentials.

## Cross-cloud ACR use by Container Apps

_Azure CLI `2.86.0`._

`az containerapp create` now supports Azure Container Registry references in
other Azure clouds rather than assuming the default cloud.

## Default Container Apps workload-profile name

_Azure CLI `2.85.0`._

`az containerapp env workload-profile add` now supplies a default profile
name when one is not specified.

## ETag-guarded AKS operations

_Azure CLI `2.69.0`._

`az aks create`, `az aks update`, and `az aks delete` accept `--if-match` and
`--if-none-match`, allowing callers to make cluster changes conditional on an
ETag instead of racing concurrent updates.

## New Container Apps and Maps defaults

_Azure CLI `2.84.0`._

`az containerapp job create` now supplies defaults for `--parallelism` and
`--replica-completion-count`. `az maps account create` likewise supplies a
default for `--sku`; pass these options explicitly when automation must not
depend on CLI defaults.

## Pod Security Standards for AKS safeguards

_Azure CLI `2.81.0`._

`az aks safeguards` adds `--pss-level` for configuring Pod Security Standards.
`az aks safeguards create` also rejects duplicate resource creation during
CLI validation.

## Registry inspection without a configured server

_Azure CLI `2.79.0`._

`az containerapp registry show` now handles container apps that have no
registry server instead of failing with a `NoneType` error.
