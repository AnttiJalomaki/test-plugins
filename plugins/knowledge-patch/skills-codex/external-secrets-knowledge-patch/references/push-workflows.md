# Push workflows

## PushSecret source, template, and mappings

A namespaced `PushSecret` accepts exactly one source under `selector`: either a
Kubernetes Secret or a `generatorRef`. `template` and `templateFrom` construct
outgoing properties before mapping. Each `data[].match` maps a source or
templated `secretKey` to a provider `remoteKey` and optional `property`
(`push-secrets`).

`updatePolicy: Replace` permits overwrites. `deletionPolicy` defaults to
`None`; set it to `Delete` when removing the `PushSecret` should clean up the
provider secret. A referenced `SecretStore` receives a finalizer when
`deletionPolicy: Delete` is used so deletion can finish safely (since 0.20.0).

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: app-credentials
spec:
  updatePolicy: Replace
  deletionPolicy: Delete
  refreshInterval: 1h
  secretStoreRefs:
    - name: destination
      kind: SecretStore
  selector:
    secret:
      name: source-credentials
  template:
    data:
      normalized: '{{ index . "raw-key" | toString | upper }}'
    templateFrom:
      - configMap:
          name: push-fragment
          items:
            - key: config.yml
  data:
    - match:
        secretKey: normalized
        remoteRef:
          remoteKey: app-credentials
          property: normalized
```

## Bulk expansion with dataTo

`PushSecret.spec.dataTo` expands every key or a regexp-filtered subset from the
Kubernetes Secret selected by `spec.selector` (since 2.3-datato). Each entry
requires a `storeRef` by name or label selector, and that store must also be in
`secretStoreRefs`.

```yaml
spec:
  secretStoreRefs:
    - name: target-store
  dataTo:
    - storeRef:
        name: target-store
      match:
        regexp: '^APP_'
      rewrite:
        - regexp:
            source: '^APP_(.*)$'
            target: '$1'
```

- Without `remoteKey`, each matched key becomes a separate provider secret or
  variable. With `remoteKey`, all matches are bundled into one JSON object.
- An absent or empty match selects all keys.
- Per-key mode supports regexp and template rewrites, but not merge. Bundle
  mode applies no rewrites.
- Configure provider `metadata` and `conversionStrategy` per entry.
- Processing order is template, key conversion, match, then rewrite.
- Explicit `spec.data` wins for the same original, unconverted Kubernetes key.
- An invalid match regexp is an error; no matches is a successful no-op.
- Duplicate remote keys within or across entries fail reconciliation and
  identify the conflicting sources.
- `IfNotExists` applies to each expanded target.
- `DeletionPolicy=Delete` records expanded targets in
  `status.syncedPushSecrets` for removal when the source Secret is deleted.

## ClusterPushSecret fan-out

A `ClusterPushSecret` creates its embedded `pushSecretSpec` in every namespace
matching any ORed `namespaceSelectors` entry (`push-secrets`).
`pushSecretName` defaults to the parent name, `pushSecretMetadata` is copied to
children, and `refreshTime` controls fan-out checks. Name collisions populate
`failedNamespaces`.

`Ready` on the cluster-scoped parent reports only child provisioning. Inspect
each generated `PushSecret` for provider synchronization failures.

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterPushSecret
metadata:
  name: app-credentials
spec:
  pushSecretName: app-credentials-push
  pushSecretMetadata:
    labels:
      managed-by: cluster-push
  namespaceSelectors:
    - matchLabels:
        team: payments
    - matchLabels:
        shared-secrets: "true"
  refreshTime: 1m
  pushSecretSpec:
    updatePolicy: Replace
    deletionPolicy: Delete
    refreshInterval: 1h
    secretStoreRefs:
      - name: destination
        kind: SecretStore
    selector:
      secret:
        name: source-credentials
```

A `ClusterPushSecret` can select and push all Secrets from a namespace instead
of selecting each individually (since 0.15.0). Cross-namespace pushes work
with `ClusterSecretStore` (since 2.1.0).

## AWS Secrets Manager push behavior

- Tags can be updated, patched, and deleted throughout their lifecycle (since
  0.19.0).
- Empty resource policies are handled during push operations (since 2.3.0).
- Tags and resource policies synchronize even when the secret value is
  unchanged, so metadata-only updates do not require a value change (since
  2.2.0).
- Equivalent resource policies are canonicalized to sorted JSON before
  comparison, avoiding ordering-only changes (since 1.2.0).
- `replicationLocations` configures replicated secret locations (since 2.7.0).

## GCP Secret Manager push behavior

- Location and replication settings are applied for pushes (since 0.17.0),
  except regional push operations omit replication settings (since 0.18.0).
- Push handling checks that a secret version exists rather than treating a
  versionless secret as usable (since 1.1.0).
- Multiple `replicationLocations` are supported (since 2.4.0).

## Kubernetes provider push behavior

- The provider implements `SecretExists` (since 2.1.0).
- It can delete an entire Secret rather than requiring key-by-key deletion
  (since 1.1.0).
- A push replaces the whole remote Secret rather than merging; existing keys
  absent from the source are removed (since 2.7.0).

## Vault and OpenBao-compatible push behavior

Vault implements the remote existence check and set operations required by
`PushSecret` (since 0.20.0).

## 1Password push behavior

- 1Password Connect is classified read-write, enabling write operations (since
  0.18.0).
- The SDK provider supports multi-field pushes and completes its push
  implementation (since 2.3.0).
- `IfNotExists` is honored (since 2.8.0).

## Other provider push behavior

- Akeyless is classified read-write so pushes route to it (since 2.7.0).
- Azure pushes can set `contentType` (since 2.4.0).
- Delinea Secret Server supports `PushSecret` (since 2.3.0).
- Infisical supports `PushSecret`; an HTTP 404 becomes `NoSecretErr`, so
  absence follows missing-secret semantics (since 2.7.0).
- Conjur returns explicit errors for unimplemented `PushSecret` and
  `DeleteSecret` operations (since 2.4.0).
- BeyondTrust can create secrets through API v3.2 (since 2.5.0).
- `PushSecret` deletion removes the intended key rather than a different key
  (since 1.3.0).
- Updating a GitHub organization secret preserves selected repositories (since
  2.7.0); `GithubProvider.orgSecretVisibility` controls organization-secret
  visibility (since 2.3.0).
