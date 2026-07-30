# Helm and Platform Operations

Use this reference for chart composition, ServiceAccounts, Pod security and
scheduling, NetworkPolicies, component scope, and OpenShift installation.

## Chart composition

The chart accepts an `enabled` value from 1.17, allowing a parent chart to turn
cert-manager on or off when it is included as a dependency.

Configured image pull secrets are applied to Deployments even when the chart is
set not to create ServiceAccounts (since 1.17).

Both keys and values in chart-managed ServiceAccount annotations are evaluated
with Helm `tpl` from 1.17. Workload-identity annotation names and values can
therefore derive from other chart values.

PodDisruptionBudget `minAvailable` and `maxAvailable` accept percentages from
1.17:

```yaml
podDisruptionBudget:
  minAvailable: "50%"
```

## Controller scope

From 1.18, starting the controller with `--namespace=<namespace>` both limits
cert-manager to that namespace and disables cluster-scoped controllers. Do not
expect ClusterIssuer or other cluster-scoped reconciliation in this mode.

## Network policy

Default chart network policies include IPv6 rules from 1.19, so dual-stack and
IPv6-only clusters do not need patched cert-manager traffic rules.

From 1.20, the chart can create NetworkPolicy resources for every cert-manager
Deployment, providing chart-managed isolation for all deployed components.

## Scheduling, user namespaces, and runtimes

`global.nodeSelector` applies a common selector to all chart components from
1.19. Use 1.19.2 or later, which correctly merges it with per-component
settings.

```yaml
global:
  nodeSelector:
    kubernetes.io/os: linux
```

On Kubernetes 1.33 or later, the experimental `global.hostUsers: false` value
places all cert-manager Pods in Kubernetes user namespaces. It is unset by
default to retain compatibility with earlier Kubernetes releases.

```yaml
global:
  hostUsers: false
```

Runtime classes are configurable for components and ACME HTTP-01 solver Pods
from 1.21. The chart-wide solver value is:

```yaml
acmesolver:
  runtimeClassName: gvisor
```

## Container identity

In 1.20, the default container UID changed from `1000` to `65532` and the GID
changed from `0` to `65532`. Update admission policies, security checks, and
volume ownership that depend on the previous identities.

## RBAC changes

The 1.21 chart no longer creates the Role and RoleBinding that allow the
controller to request tokens for its own ServiceAccount. Issuers selecting that
account through `serviceAccountRef.name`, including some Vault Kubernetes auth
and Route53 configurations, require explicit RBAC or a dedicated account.

The `global.rbac.disableHTTPChallengesRole` chart value was added in 1.18.0 and
removed in 1.18.2 due to a bug. It is unavailable for the remainder of 1.18.

From 1.19.6, the aggregate `cert-manager-edit` ClusterRole restricts direct
Challenge and Order manipulation. Certificate-driven issuance is unchanged;
grant explicit RBAC to tools that directly manage those internal resources.

## Startup API check cleanup

Set the opt-in 1.21 value `startupapicheck.ttlSecondsAfterFinished` to let the
Kubernetes TTL-after-finished controller delete the completed startup API check
Job.

## OpenShift and OperatorHub

OperatorHub publication ends at cert-manager 1.16.5 in both Red Hat OpenShift
and community catalogs. OperatorHub installations need another distribution
method for 1.17 or later.

Do not deploy cert-manager 1.20.0 on OpenShift. It omitted issuer-finalizer RBAC
required by the Order controller; 1.20.1 restores the rule.

At the support-lifecycle snapshot, cert-manager 1.21 maps to OpenShift
4.20-4.22 and Kubernetes 1.33-1.36, while cert-manager 1.20 maps to OpenShift
4.19-4.21 and Kubernetes 1.32-1.35. Mappings for OpenShift releases that do not
yet exist may be predictions.

## Webhook operation

Use 1.20.2 or later when both `webhook.config` and `webhook.volumes` are set;
earlier versions can render invalid Helm YAML.

From 1.21, wall-clock polling lets the webhook detect a missed serving
certificate renewal after system suspend or VM live migration and recover
within one minute of resume.
