# Notifications, Receivers, and traces

## Event metadata

Since 2.5.0, annotations on Flux Kustomization and HelmRelease objects can add
metadata to notification events. Keys use the
`event.toolkit.fluxcd.io/` prefix.

An image policy can update `event.toolkit.fluxcd.io/image` together with the
workload image value, allowing providers to include the full new image
reference in the message body.

## Git commit statuses

Since 2.5.0, notification-controller can update Git commit statuses for events
from Kustomizations backed by `OCIRepository` sources.

Since 2.6.0, a notification Provider can derive the status identifier with
the CEL-based `.spec.commitStatusExpr`. Use this to distinguish clusters or
workloads in a monorepo:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: github-status
  namespace: flux-system
spec:
  type: github
  address: https://github.com/my-gh-org/my-gh-repo
  secretRef:
    name: github-app-auth
  commitStatusExpr: "(event.involvedObject.kind + '/' + event.involvedObject.name + '/' + event.metadata.clusterName)"
```

Since 2.8.0, commit-status events can originate from any Flux API, including
HelmRelease. Add the commit annotation for status providers:

```yaml
metadata:
  annotations:
    event.toolkit.fluxcd.io/commit: "<commit-sha>"
```

## Pull and merge request comments

Since 2.8.0, these Provider types directly post and update one deduplicated
deployment-status comment:

- `githubpullrequestcomment`
- `gitlabmergerequestcomment`
- `giteapullrequestcomment`

They do not require an intermediary CI workflow. Comment providers read the
change-request annotation:

```yaml
metadata:
  annotations:
    event.toolkit.fluxcd.io/change_request: "42"
```

## Provider authentication and transport

Since 2.6.0:

- Azure Event Hub publishing supports Azure Workload Identity.
- `github` and `githubdispatch` Providers support GitHub App authentication.

Since 2.7.0, Provider object-level Workload Identity extends to Azure DevOps,
Azure Event Hub, and Google Pub/Sub through
`Provider.spec.serviceAccountName`.

Providers can read proxy credentials from `spec.proxySecretRef` and mTLS
material from `spec.certSecretRef`. Zulip is also a supported Provider type.

## OpenTelemetry reconciliation traces

Since 2.7.0, a Provider of type `otel` turns Flux events into related spans.
Source objects create root spans; consuming Kustomizations and HelmReleases
create child spans:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: jaeger
  namespace: flux-system
spec:
  type: otel
  address: http://jaeger-collector.jaeger:4318/v1/traces
```

## Receiver filtering and reconciliation

Since 2.5.0, the Receiver API can filter declared resources with a CEL
expression, limiting webhook reconciliation to matching objects.

Since 2.7.0, notification-controller can immediately reconcile after a
Receiver `secretRef` changes. Opt in the referenced Secret with:

```yaml
metadata:
  labels:
    reconcile.fluxcd.io/watch: Enabled
```

Or use `--watch-configs-label-selector` on the controller.

Since 2.9.0, generic Receivers can validate an OIDC ID token instead of an
HMAC shared secret. Invoke a Receiver from the CLI without manually
constructing its webhook request:

```shell
flux trigger receiver
```

When upgrading GCR Receivers to 2.9.0, add `email` and `audience` to the
referenced Secret.

