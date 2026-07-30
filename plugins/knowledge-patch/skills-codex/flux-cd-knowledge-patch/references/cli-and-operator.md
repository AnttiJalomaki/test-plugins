# CLI and Flux Operator

## Debug merged configuration

Since 2.5.0, these commands print the effective values after merging inline
configuration with referenced ConfigMaps and Secrets:

```bash
flux debug kustomization --show-vars
flux debug helmrelease --show-values
```

Referenced Secret values are printed in clear text. Treat the terminal,
captured output, logs, and pasted diagnostics as sensitive.

## Stable OCI artifact commands

Since 2.6.0, these artifact commands are stable:

- `flux build artifact`
- `flux push artifact`
- `flux pull artifact`
- `flux tag artifact`
- `flux diff artifact`
- `flux list artifacts`

The associated stable media types are
`application/vnd.cncf.flux.config.v1+json` for configuration and
`application/vnd.cncf.flux.content.v1.tar+gzip` for content.

## CLI plugins

Since 2.9.0, the Flux CLI installs independently versioned plugins under
`~/fluxcd/plugins` and exposes each one as `flux <plugin>`. The initial catalog
includes Mirror for declarative registry mirroring and Schema for JSON Schema
and CEL validation.

```bash
flux plugin search
flux plugin install schema@0.5.0
flux plugin list
flux plugin update schema
flux plugin uninstall schema
```

Pin plugin versions or immutable digests in reproducible automation.

## Receiver triggering

Since 2.9.0, invoke a Receiver without hand-building its webhook request:

```bash
flux trigger receiver
```

Generic Receivers can authenticate these requests by validating an OIDC ID
token instead of an HMAC shared secret.

## Flux Operator Web UI

The Flux Operator Web UI added by 2.8.0 provides cluster and GitOps-resource
monitoring, rollout inspection, delivery graphs, and RBAC-guarded actions. It
supports OIDC single sign-on together with Kubernetes RBAC for multi-tenant
clusters.

Its 2.9.0 additions include a workload dashboard for Deployments, StatefulSets,
DaemonSets, and CronJobs, plus a multi-pod, multi-container log viewer.
Workload actions and log access use Kubernetes RBAC through user impersonation.
