# Helm, Operations, and Security

Use this reference for chart composition, scheduling, Pod identity, RBAC,
NetworkPolicy, controller scope, and operational cleanup.

## Chart composition and templating

### Dependency toggle

Since `1.17`, the chart accepts an `enabled` value. A parent chart can use it to
toggle cert-manager when declaring cert-manager as a dependency.

### Templated ServiceAccount annotations

Since `1.17`, both keys and values in chart-managed ServiceAccount annotations
are evaluated with Helm's `tpl` function. Workload-identity annotation names
and values can therefore derive from other chart values. Treat untrusted chart
values as template input rather than plain strings.

### Pull secrets without managed ServiceAccounts

Since `1.17`, configured image pull secrets are attached to Deployments even
when the chart is configured not to create ServiceAccounts.

### Percentage PodDisruptionBudgets

Since `1.17`, `podDisruptionBudget.minAvailable` and
`podDisruptionBudget.maxAvailable` accept percentages:

```yaml
podDisruptionBudget:
  minAvailable: "50%"
```

Set only the field appropriate for the disruption policy.

### Webhook config and volumes

Use 1.20.2 or later when both `webhook.config` and `webhook.volumes` are set.
Earlier releases can render invalid Helm YAML for that combination.

## Scheduling and isolation

### Common node selector

The `global.nodeSelector` value in `1.19` applies a common selector to all
cert-manager components. Use 1.19.2 or later, which correctly merges it with
component-level settings:

```yaml
global:
  nodeSelector:
    kubernetes.io/os: linux
```

### Kubernetes user namespaces

On Kubernetes 1.33 or later, the experimental setting below configures all
chart-managed Pods to use Kubernetes user namespaces:

```yaml
global:
  hostUsers: false
```

It is unset by default to keep the `1.19` chart compatible with Kubernetes
versions before 1.33.

### Runtime classes

In `1.21`, runtime classes can be configured for cert-manager components and
HTTP-01 solver Pods. The solver chart value is:

```yaml
acmesolver:
  runtimeClassName: gvisor
```

### Container identity defaults

In `1.20`, the default container UID changed from `1000` to `65532`, and the
default GID changed from `0` to `65532`. Update admission policy, security
context assertions, and volume ownership assumptions before rollout.

## NetworkPolicy

### IPv6 defaults

Since `1.19`, the chart's default network policy contains IPv6 rules. Dual-stack
and IPv6-only clusters no longer need a locally patched policy for ordinary
cert-manager traffic.

### Policies for every Deployment

In `1.20`, the chart can create NetworkPolicy resources for every cert-manager
Deployment. Enable this when chart-managed network isolation is desired, then
validate DNS, Kubernetes API, issuer endpoint, webhook, and monitoring flows.

## RBAC changes

### HTTP challenge role value is unavailable

`global.rbac.disableHTTPChallengesRole` was added in 1.18.0 and removed in
1.18.2 because of a bug. Do not configure it for the remainder of the 1.18
line.

### Direct Challenge and Order access

Starting in 1.19.6, the aggregate `cert-manager-edit` ClusterRole no longer
allows creation of Challenges or creation, patching, and updating of Orders.
Normal Certificate-driven issuance is unaffected. Grant dedicated permissions
to tooling that intentionally manages these internal resources.

### Controller ServiceAccount token creation

In `upgrade-1.21`, the chart stopped creating the Role and RoleBinding that let
the controller create tokens for its own ServiceAccount. When an issuer's
`serviceAccountRef.name` points to that ServiceAccount, create the RBAC
explicitly or migrate to a dedicated ServiceAccount with its own permissions.
Vault Kubernetes auth and Route53 are common affected configurations.

### OpenShift Order-controller fix

Version 1.20.0 omitted issuer-finalizer RBAC required by the Order controller
on OpenShift. Version 1.20.1 restored it; OpenShift installations should use
1.20.1 or later.

## Controller scope and finalizers

### Namespace-scoped operation

Since `1.18`, `--namespace=<namespace>` limits cert-manager to that namespace
and disables cluster-scoped controllers. Do not expect ClusterIssuer or other
cluster-scoped reconciliation in this mode.

### Domain-qualified finalizer

`UseDomainQualifiedFinalizer` became beta and enabled by default in `1.17`.
The domain-qualified finalizer avoids Kubernetes warnings and does not require
manual feature-gate enablement.

## Completed Job cleanup

In `1.21`, set the opt-in Helm value
`startupapicheck.ttlSecondsAfterFinished` to let Kubernetes' TTL-after-finished
controller delete the completed startup API check Job.
