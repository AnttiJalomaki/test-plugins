# PushSecret Workflows

Use this reference when sending Kubernetes Secret data or generated data to a
provider. Confirm write support per provider; read support and maturity do not
imply push, merge, existence-check, or deletion support.

## Namespaced PushSecret model

A namespaced `PushSecret` accepts exactly one source under `spec.selector`: a
Kubernetes Secret or a `generatorRef`. `template` and `templateFrom` build the
outgoing values before mapping. Each `data[].match` maps a source or templated
`secretKey` to a provider `remoteKey` and an optional property (push-secrets).

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
    - match:
        secretKey: config.yml
        remoteRef:
          remoteKey: app-config
          property: config.yml
```

`updatePolicy: Replace` permits overwrite. `deletionPolicy` defaults to `None`;
use `Delete` when the provider secret must be removed with the `PushSecret`.
Correct deletion removes the intended remote key (1.3.0). A referenced
`SecretStore` receives a finalizer when `deletionPolicy: Delete` so remote
cleanup can complete before the store disappears (0.20.0).

## ClusterPushSecret fan-out

A `ClusterPushSecret` can push all Secrets from a namespace rather than naming
each one individually (0.15.0). In its current fan-out form, it embeds a
`pushSecretSpec` and creates a child in every namespace matched by any entry in
the ORed `namespaceSelectors` list (push-secrets).

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
    data:
      - match:
          secretKey: password
          remoteRef:
            remoteKey: app-credentials
            property: password
```

`pushSecretName` defaults to the parent name, `pushSecretMetadata` is copied to
children, and `refreshTime` controls fan-out checks. Name collisions appear in
`failedNamespaces`. The parent `Ready` condition says that children were
provisioned; it does not say that their provider writes succeeded. Inspect each
generated child for remote-sync status (push-secrets).

Cross-namespace pushes through `ClusterSecretStore` work (2.1.0), subject to
store access conditions, namespace validation, and the provider's own write
support.

## Bulk expansion with dataTo

`PushSecret.spec.dataTo` expands every key or a regexp-filtered subset from the
Kubernetes Secret selected by `spec.selector` (2.3-datato). This avoids one
`spec.data` entry per key.

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

Each `dataTo` entry requires a `storeRef` by name or label selector, and the
selected store must also occur in `secretStoreRefs`.

### Per-key versus bundle mode

- Without `remoteKey`, every match becomes a separate provider secret or
  variable. Per-key entries can apply regexp and template rewrites, but cannot
  use merge rewrites.
- With `remoteKey`, all matches are serialized into one JSON object at that
  remote key. Bundle mode does not apply rewrites.
- An absent or empty match selects every key.
- Provider-specific `metadata` and `conversionStrategy` are set per entry.

### Ordering, conflicts, and lifecycle

Expansion order is significant (2.3-datato):

1. Apply `spec.template`.
2. Convert keys.
3. Match converted keys.
4. Rewrite matched keys in per-key mode.

An explicit `spec.data` item wins when it targets the same original,
unconverted Kubernetes key. Invalid match regexps put the resource in an error
state. No matches is a successful no-op. Duplicate remote keys within an entry
or across entries fail reconciliation and identify the conflicting sources.

`updatePolicy: IfNotExists` is enforced for each expanded target.
`deletionPolicy: Delete` records every expanded target in
`status.syncedPushSecrets` so all of them can be removed after the source Secret
is deleted.

## Provider write and deletion behavior

### AWS Secrets Manager

- AWS tags can be read and used in secret workflows (0.16.0); tag update, patch,
  and delete lifecycle operations are supported (0.19.0).
- Tags and resource policies synchronize even if the secret value has not
  changed, so metadata-only updates are effective (2.2.0).
- Resource policies are canonicalized to sorted JSON before comparison, so key
  ordering alone does not produce a false change (1.2.0).
- An empty resource policy is handled during push (2.3.0).
- `replicationLocations` configures replicated AWS secrets (2.7.0).

### Google Secret Manager

- Location and replication settings are applied to pushes correctly (0.17.0).
- Regional pushes intentionally omit replication settings (0.18.0).
- A push checks whether a secret version exists; a secret with no version is not
  treated as a usable target (1.1.0).
- Regional existence checks honor the configured store location (1.2.0).
- Multiple `replicationLocations` are supported (2.4.0).

### Kubernetes provider

- The provider can delete a whole Secret instead of deleting keys one by one
  (1.1.0).
- `SecretExists` is implemented (2.1.0).
- A push replaces the entire remote Secret rather than merging into it; keys
  absent from the pushed source are removed (2.7.0).

### 1Password

- 1Password Connect is classified as a read-write store (0.18.0).
- The SDK provider supports multi-field pushes and completes its push
  implementation (2.3.0).
- 1Password honors `IfNotExists` (2.8.0).

### Vault, Delinea, Infisical, and Akeyless

- Vault implements the remote existence check and set operations needed by
  `PushSecret` (0.20.0).
- Delinea Secret Server supports push (2.3.0).
- Infisical supports push and maps HTTP 404 to `NoSecretErr`, so absence follows
  the normal missing-secret path (2.7.0).
- Akeyless is classified read-write, allowing push operations to route to it
  (2.7.0).

### Azure, GitHub, and Conjur

- Azure push accepts `contentType` (2.4.0).
- GitHub organization secrets accept `orgSecretVisibility` (2.3.0), and updates
  preserve the selected repositories (2.7.0).
- Conjur explicitly errors for its unimplemented `PushSecret` and `DeleteSecret`
  operations instead of appearing to succeed (2.4.0).
