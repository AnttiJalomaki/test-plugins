# Notifications, observability, and Receivers

## Contents

- [Event metadata and image updates](#event-metadata-and-image-updates)
- [Git commit statuses](#git-commit-statuses)
- [Provider authentication and transport](#provider-authentication-and-transport)
- [OpenTelemetry reconciliation traces](#opentelemetry-reconciliation-traces)
- [Pull and merge request comments](#pull-and-merge-request-comments)
- [Receiver targeting and authentication](#receiver-targeting-and-authentication)

## Event metadata and image updates

Since 2.5.0, annotations on Flux Kustomization and HelmRelease objects can add
metadata to notification events. An image-policy marker can update
`event.toolkit.fluxcd.io/image` alongside the workload value, so providers
receive the new full image reference in the message body.

## Git commit statuses

Notification-controller can update Git commit statuses for events from
Kustomizations backed by OCIRepository sources (since 2.5.0).

Since 2.6.0, a notification `Provider` can derive a status identifier with the
CEL-based `.spec.commitStatusExpr`. This lets monorepo fleets distinguish each
cluster:

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

Since 2.8.0, commit-status events may come from any Flux API, including
HelmRelease. Put the commit SHA in the
`event.toolkit.fluxcd.io/commit` annotation.

## Provider authentication and transport

Since 2.6.0:

- Notification-controller can use Azure Workload Identity to publish to Azure
  Event Hub.
- `github` and `githubdispatch` Providers can authenticate as a GitHub App.

Since 2.7.0, `Provider.spec.serviceAccountName` supports Workload Identity for
Azure DevOps, Azure Event Hub, and Google Pub/Sub. Providers can read proxy
credentials from `spec.proxySecretRef` and mutual-TLS material from
`spec.certSecretRef`. Zulip is also a supported Provider type.

## OpenTelemetry reconciliation traces

An `otel` Provider turns Flux events into related spans (since 2.7.0). Source
objects create root spans; consuming Kustomizations and HelmReleases create
child spans.

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

## Pull and merge request comments

Since 2.8.0, `githubpullrequestcomment`, `gitlabmergerequestcomment`, and
`giteapullrequestcomment` Providers post and update a deduplicated deployment
status comment directly, without an intermediary CI workflow.

Use `event.toolkit.fluxcd.io/change_request` for comment Providers and
`event.toolkit.fluxcd.io/commit` for status Providers:

```yaml
metadata:
  annotations:
    event.toolkit.fluxcd.io/change_request: "42"
    event.toolkit.fluxcd.io/commit: "<commit-sha>"
```

## Receiver targeting and authentication

Since 2.5.0, a `Receiver` can filter its declared resources with CEL, limiting
webhook reconciliation to matching objects.

Since 2.7.0, notification-controller can immediately reconcile a Receiver when
its referenced Secret changes. Label that Secret:

```yaml
metadata:
  labels:
    reconcile.fluxcd.io/watch: Enabled
```

Alternatively, select watched references through the controller's
`--watch-configs-label-selector` option.

Since 2.9.0, generic Receivers can validate an OIDC ID token rather than an
HMAC shared secret. `flux trigger receiver` invokes a Receiver without manually
constructing the webhook request.

At the same upgrade boundary, GCR Receivers must add `email` and `audience` to
their referenced Secret.
