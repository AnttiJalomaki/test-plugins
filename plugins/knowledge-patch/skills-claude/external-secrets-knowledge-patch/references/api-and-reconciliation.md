# API Resources and Reconciliation

Use this reference for `ExternalSecret`, `ClusterExternalSecret`, store, target,
status, refresh, validation, and namespace behavior. Provider-specific fields are
in the provider references; outgoing-secret behavior is in `push-secrets.md`.

## API version and schema transitions

- Provider examples use `apiVersion: external-secrets.io/v1` (0.17.0). Prefer
  the stable API and check the installed CRD rather than copying an older beta
  example.
- Serving the legacy beta API can be toggled during migration (1.3.0). This is a
  transition control, not a reason to leave manifests on the beta version.
- `ExternalSecretRewrite` has validation constraints (0.19.0), and invalid
  rewrite definitions are rejected at admission instead of reaching reconcile.
- Namespace values in namespaced `secretRef` configurations are validated
  (0.20.0). Do not rely on a later provider error for an invalid cross-namespace
  reference.
- `generatorRef` validates `externalsecret_type` (1.0.0). A reference that used
  to bypass that check must be corrected to a supported type.
- CRDs expose selectable fields (0.20.0), so supported resource fields can be
  queried with Kubernetes field selectors.

## Refresh policies and state

`Periodic` is the default policy. A zero `refreshInterval` fetches and creates
the target once, but performs no later periodic updates. `OnChange` ignores the
interval and reacts only to changes in the `ExternalSecret` metadata or spec.

`CreatedOnce` is stateful: its status lets ESO restore a target Secret that was
changed or deleted while the same `ExternalSecret` survives. Deleting and
recreating the `ExternalSecret` loses that status, so ESO can overwrite a target
that still exists. Creation policy does not prevent this recreation-time write
(api-v1).

For a generated credential that must survive deletion and never be replaced:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: bootstrap-credential
spec:
  refreshPolicy: CreatedOnce
  secretStoreRef:
    name: app-store
    kind: SecretStore
  target:
    name: bootstrap-credential
    creationPolicy: Orphan
    immutable: true
  data:
    - secretKey: password
      remoteRef:
        key: app/password
```

Additional timing behavior:

- An `ExternalSecret` can define sync windows that gate periodic refreshes
  (2.7.0). A healthy object can therefore remain unchanged outside its window.
- `SecretStore.refreshInterval` accepts duration strings (2.8.0).
- Failed reconciliations use a much less aggressive retry cadence than earlier
  behavior (0.14.0). Do not infer a stalled controller from the lack of rapid
  retries.
- SecretStore reconciliation itself can be enabled or disabled by a controller
  flag (1.2.0). Check this flag when stores never update status.

## Manual refresh

The annotation names differ by resource (api-v1):

```sh
kubectl annotate es my-es force-sync=$(date +%s) --overwrite
kubectl annotate ces my-ces external-secrets.io/force-sync=$(date +%s) --overwrite
```

An `ExternalSecret` responds only when its refresh policy supports the action.
For a `ClusterExternalSecret`, setting, changing, or deleting
`external-secrets.io/force-sync` propagates the change to every owned child.

## Target creation and content

- An `ExternalSecret` source can select a dynamic target rather than requiring a
  target fixed in advance (1.0.0).
- `CreateOrMerge` is accepted as a target creation policy (2.8.0).
- Certificate processing accepts PKCS#12 bundles containing certificates but no
  private key (0.20.0).
- `objectMeta` and `ownerReferences` propagate to target resources (2.3.0).

### Metadata copying is replaced by template metadata

Labels and annotations on the `ExternalSecret` are normally copied to its target
Secret. Once `target.template.metadata` configures a category, that template
metadata replaces the implicit copy. Empty maps explicitly suppress copying
(api-v1):

```yaml
spec:
  target:
    template:
      metadata:
        labels: {}
        annotations: {}
```

See `templates-generators-and-cli.md` for data selection, delimiters, native
values, functions, renderer commands, `templateFrom`, and generator behavior.

## ClusterExternalSecret fan-out

`ClusterExternalSecret.spec.namespaceSelectors` is a list and its entries are
ORed. The singular `namespaceSelector` and the explicit `namespaces` field are
deprecated. If the desired child name already exists, ESO reports that namespace
as failed rather than taking ownership of the object (api-v1).

```yaml
spec:
  namespaceSelectors:
    - matchLabels:
        team: payments
    - matchLabels:
        shared-credentials: "true"
```

Every generated child normally polls the upstream provider independently, so
provider calls grow linearly with namespaces. To fetch once, create one source
`ExternalSecret` in a dedicated namespace, expose its resulting Secret through a
Kubernetes-provider `ClusterSecretStore`, and have the `ClusterExternalSecret`
replicate through that store. Only the source object then contacts the upstream
provider (api-v1).

The Helm chart gates write permissions for `externalsecrets` on
`processClusterExternalSecret` (2.5.0). Disabling that controller no longer
leaves the write permission enabled.

## ClusterSecretStore access conditions

A `ClusterSecretStore` can restrict which namespaces may reference it with
`spec.conditions`. Label selectors, explicit namespace names, and regular
expressions are separate ORed alternatives; satisfying any one grants access
(api-v1).

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

Namespaced `ExternalSecret` and `SecretStore` objects still cannot reference a
different namespaced store, Secret, or other namespaced referent. A
cluster-scoped resource needs a separate scope and authorization review.

## Store status, lifecycle, and retries

- A `SecretStore` can report unknown status when its state cannot be determined
  (0.20.0). Preserve the distinction between unknown and a known failure.
- A store can be marked deprecated so operators can identify it as a migration
  target (1.2.0).
- A store backed by an unmaintained provider produces warning events from both
  the controller and admission webhook. The annotation below suppresses only
  the controller warning; the admission warning cannot be disabled (api-v1):

```yaml
metadata:
  annotations:
    external-secrets.io/ignore-maintenance-checks: "true"
```

A namespaced `SecretStore` can configure HTTP retries (api-v1):

```yaml
spec:
  retrySettings:
    maxRetries: 5
    retryInterval: 10s
```

The v1 API documentation lists these retry settings only for AWS, HashiCorp
Vault, IBM, and Doppler providers. Do not assume another provider consumes them.

## Webhook and controller-visible behavior

- Webhook-provider requests include the `ExternalSecret` namespace, enabling a
  namespace-aware endpoint (0.20.0).
- `ValidatingWebhookConfiguration` accepts annotations (0.16.0).
- The validating webhook obtains `failurePolicy` for `SecretStore` dynamically
  (2.0.0), and the chart applies its configured policy to the
  `ClusterSecretStore` webhook (2.4.0).
- Tabular output includes `storeType` (0.14.0).
- `ExternalSecret` and `PushSecret` printer output includes a Last Sync column
  (2.2.0).
- The controller logs target Secret deletion and data-key changes (2.7.0), which
  helps distinguish intended reconcile effects from external mutation.
