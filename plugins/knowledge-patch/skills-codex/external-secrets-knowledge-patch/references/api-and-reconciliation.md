# API and reconciliation

## ExternalSecret refresh and target lifecycle

- `Periodic` is the default refresh policy. With `refreshInterval: 0`, it
  fetches and creates once but does not update later. `OnChange` ignores the
  interval and reacts only to `ExternalSecret` metadata or spec changes.
  `CreatedOnce` repairs a target changed or deleted while the same object
  survives, but recreating the `ExternalSecret` resets status and can rewrite
  the target. Creation policies do not prevent that recreation-time rewrite
  (`api-v1`).
- For a generated credential that must survive deletion without replacement,
  use `CreatedOnce` with `target.creationPolicy: Orphan` and
  `target.immutable: true` (`api-v1`).
- `ExternalSecret` supports sync windows that gate periodic refreshes (since
  2.7.0).
- `SecretStore.refreshInterval` accepts duration strings (since 2.8.0).
- The controller can enable or disable `SecretStore` reconciliation with a
  flag (since 1.2.0).
- Failed reconciliations use a much less aggressive retry cadence than before
  (since 0.14.0).

## Target construction and ownership

- Sources support dynamic targets rather than requiring the target to be fixed
  in advance (since 1.0.0).
- `ExternalSecret` accepts `CreateOrMerge` as a target creation policy (since
  2.8.0).
- `objectMeta` and `ownerReferences` propagate to target resources (since
  2.3.0).
- Secret templates can add finalizers to generated Secrets (since 0.20.0).
- Sources have a configurable null-byte policy (since 2.3.0).
- PKCS#12 handling accepts certificate-only bundles without a private key
  (since 0.20.0).

## Target metadata

Labels and annotations on an `ExternalSecret` normally copy to its target
Secret. Once `target.template.metadata` is configured, it replaces that
implicit behavior. Empty maps suppress copying for that metadata kind
(`api-v1`):

```yaml
spec:
  target:
    template:
      metadata:
        labels: {}
        annotations: {}
```

## ClusterExternalSecret selection and fan-out

- Use the plural `spec.namespaceSelectors`; its entries are ORed. The singular
  `namespaceSelector` and explicit `namespaces` fields are deprecated. A name
  collision with an existing `ExternalSecret` is reported in failed
  namespaces instead of taking the object over (`api-v1`).
- Every generated `ExternalSecret` independently polls its upstream provider,
  so calls grow linearly with matched namespaces. At scale, fetch once to a
  dedicated namespace, point a Kubernetes-provider `ClusterSecretStore` at
  that source Secret, and fan out through the Kubernetes provider
  (`api-v1`).
- Chart write permissions for `externalsecrets` are gated by
  `processClusterExternalSecret`; disabling that reconciler no longer leaves
  the write permission enabled (since 2.5.0).

```yaml
spec:
  namespaceSelectors:
    - matchLabels:
        team: payments
    - matchLabels:
        shared-credentials: "true"
```

## Manual refresh

Change the unqualified `force-sync` annotation to refresh an `ExternalSecret`
whose policy supports refresh. For `ClusterExternalSecret`, change or delete
`external-secrets.io/force-sync`; the change propagates to every owned
`ExternalSecret` (`api-v1`).

```sh
kubectl annotate es my-es force-sync=$(date +%s) --overwrite
kubectl annotate ces my-ces external-secrets.io/force-sync=$(date +%s) --overwrite
```

## Store access and namespace rules

- Namespaced `ExternalSecret` and `SecretStore` resources cannot make
  cross-namespace references to stores, Secrets, or other namespaced
  referents. Namespaces in `secretRef` are validated rather than accepted for
  later failure (since 0.20.0; clarified in `api-v1`).
- `ClusterSecretStore.spec.conditions` can restrict consumer namespaces.
  Label selectors, explicit names, and regular expressions are separate ORed
  conditions, so satisfying any one grants access (`api-v1`).
- Stores can be marked deprecated so operators can identify stores that should
  no longer be used (since 1.2.0).
- A `SecretStore` can report unknown status when its state cannot be
  determined (since 0.20.0).
- The Kubernetes provider's `auth` field is optional when no explicit
  authentication block is needed (since 0.20.0).

```yaml
spec:
  conditions:
    - namespaceSelector:
        matchLabels:
          secrets-access: "true"
    - namespaces:
        - platform-system
    - namespaceRegexes:
        - "tenant-.*"
```

## Store retries and warnings

- A namespaced `SecretStore` can set `retrySettings.maxRetries` and
  `retrySettings.retryInterval`. The v1 API documents this only for AWS,
  HashiCorp Vault, IBM, and Doppler providers (`api-v1`).
- ConfigMap access through `CAProvider` works correctly (since 2.4.0).
- Providers without an explicit maintainer generate both controller and
  admission-webhook warnings. The
  `external-secrets.io/ignore-maintenance-checks: "true"` annotation suppresses
  only the controller warning; the admission warning cannot be disabled
  (`api-v1`).

```yaml
spec:
  retrySettings:
    maxRetries: 5
    retryInterval: 10s
```

## Validation and admission behavior

- Validation constraints apply to `ExternalSecretRewrite`; invalid rewrites
  are rejected (since 0.19.0).
- `generatorRef` validates `externalsecret_type`; invalid references no longer
  bypass that check (since 1.0.0).
- Webhook provider requests include the `ExternalSecret` namespace for
  namespace-aware behavior (since 0.20.0).
- The validating webhook obtains the `SecretStore` `failurePolicy`
  dynamically (since 2.0.0).
- The chart applies `failurePolicy` to the `ClusterSecretStore` webhook (since
  2.4.0).
- `ValidatingWebhookConfiguration` resources support annotations (since
  0.16.0).
- Serving the legacy beta API version is configurable during migration (since
  1.3.0).
- Provider examples use the stable `apiVersion: external-secrets.io/v1`
  (since 0.17.0).

## Status, columns, and observability

- Tabular store output includes `storeType` (since 0.14.0).
- `ExternalSecret` and `PushSecret` printers include Last Sync (since 2.2.0).
- CRDs declare supported selectable fields for Kubernetes field selectors
  (since 0.20.0).
- Secret deletion and data-key changes are logged by the controller (since
  2.7.0).

## Value processing

- Value-scoped processing preserves native values instead of coercing them to
  strings (since 0.19.0).
- Secret metadata can explicitly request decoded-value encoding (since
  0.15.0).
- `result.jsonpath` in `dataFrom` can be templated so extraction paths can be
  selected dynamically (since 0.18.0).
