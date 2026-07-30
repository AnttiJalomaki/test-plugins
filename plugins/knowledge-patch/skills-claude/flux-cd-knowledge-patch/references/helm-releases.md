# Helm releases

## Apply and health behavior

Flux ships Helm v4 beginning with 2.8.0. New releases use server-side apply;
releases already stored by Helm remain on client-side apply until explicitly
opted in. Kstatus health is the default for all HelmReleases, and CEL
expressions can define readiness for Helm-managed objects.

Enable `UseHelm3Defaults` to retain the earlier apply and health behavior.

Beginning with 2.9.0, the post-render strategy defaults to `combined`, so Helm
hooks pass through post-renderers. Configure `nohooks` explicitly before an
upgrade if a chart relies on the old behavior.

## Retry and cancellation

Since 2.7.0, HelmRelease install and upgrade handling supports the
`RetryOnFailure` strategy.

The helm-controller `CancelHealthCheckOnNewRevision` gate, added in 2.8.0,
cancels an active health check for a new source revision, spec change,
referenced ConfigMap or Secret change, manual reconciliation, or Receiver
trigger. It reports `HealthCheckCanceled` on the `Ready` condition.

Enable `DefaultToRetryOnFailure` together with cancellation so a release does
not become stuck under the default no-retry configuration.

## Values and debugging

Since 2.9.0, `HelmRelease.valuesFrom` has a literal mode equivalent to
`helm install --set-literal`. The entire referenced ConfigMap or Secret key is
used as one string without type parsing or dotted-property expansion.

Use the debug command to inspect effective values after inline values and
referenced ConfigMaps or Secrets are merged:

```shell
flux debug helmrelease --show-values
```

This prints referenced Secret values in clear text. Protect terminal output,
logs, and captured artifacts accordingly.

## Resource inventory

Since 2.8.0, each HelmRelease records managed objects in `.status.inventory`.
Use the inventory for debugging, auditing, and tools that need the deployed
resource set.

## Dependencies and custom health

Since 2.7.0, `HelmRelease.spec.dependsOn` entries can use CEL readiness
expressions rather than relying only on the dependency's Ready condition.

Since 2.8.0, CEL expressions can define health for Helm-managed objects. Since
2.9.0, a health expression may omit `kind` to apply across all kinds in the
selected API group.

## Referenced data watches

Since 2.7.0, helm-controller can immediately reconcile when a referenced
`valuesFrom` ConfigMap or Secret, or either kubeConfig reference, changes.
Label an individual referenced object:

```yaml
metadata:
  labels:
    reconcile.fluxcd.io/watch: Enabled
```

Alternatively, use `--watch-configs-label-selector` on the controller to
select watched objects globally.

## OCI charts and generated artifacts

The helm-controller v1.3.0 `DisableChartDigestTracking` feature gate, shipped
with Flux 2.6.0, disables the default behavior of appending an OCI Helm chart
digest to its chart version.

Since 2.7.0, a HelmRelease can set `spec.chartRef.kind: ExternalArtifact` to
consume output from the optional source-watcher's `ArtifactGenerator`. Since
2.8.0, ArtifactGenerator can extract and modify Helm charts while producing
artifacts.

