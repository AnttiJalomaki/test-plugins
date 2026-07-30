# Migrations and Breaking Changes

Use this reference before changing Loki binaries, container images, Operators,
or deployment modes. Resolve each item against the versions and components
actually installed.

## Promtail to Grafana Alloy

Promtail was deprecated in 3.4.0 after its code moved into Grafana Alloy.
Alloy provides migration documentation and a configuration-conversion utility.
Promtail was then removed as of 3.7.3, so an upgrade crossing that maintenance
release requires replacing Promtail rather than merely tolerating a warning.

Lambda-promtail is explicitly outside this deprecation and removal. Do not
replace or remove a Lambda-promtail deployment solely because Promtail was
removed.

## Legacy storage, configuration, and APIs

The BoltDB store is deprecated as of 3.4.0, together with additional legacy
configuration options and API endpoints. Before upgrading:

1. Inventory every configured store, schema period, legacy option, and API
   consumer.
2. Migrate consumers away from deprecated endpoints and configuration.
3. Exercise reads across old schema periods after changing storage clients.

Deprecated ksonnet configurations were removed in 3.5.0. Treat any remaining
ksonnet-generated deployment as a required migration, not a supported input.

## Promtail image tooling

The Promtail Docker image no longer contains `wget` as of 3.4.0. Health checks,
startup scripts, debugging workflows, and derived images that invoked it must
use another available tool or explicitly add the dependency.

## Operator OTLP attribute dropping

The Loki Operator gained the ability to drop OTLP attributes in 3.5.0, and the
change is classified as breaking. Review generated configuration, label and
metadata expectations, and downstream queries before enabling or upgrading
this behavior.

## Deployment-mode and chart deprecations

Simple Scalable Deployment mode is deprecated as of 3.6.0 and scheduled for
removal before Loki 4.0. Plan a move to another supported deployment mode.

The following community charts are deprecated as of 3.6.0:

- `LGTM-distributed`
- `loki-canary`
- `loki-distributed`
- `loki-simple-scalable`

Do not confuse these deprecated charts with features of the maintained Loki
chart that have similar workload names.

## Label precedence

In 3.7.0, parsed labels no longer override structured metadata with the same
name. This is a breaking semantic change. Queries, derived fields, dashboards,
alerts, and retention rules that relied on the parsed value must be adjusted
to the structured-metadata value or avoid the collision.

## Scheduler execution

Two scheduler engine changes in 3.7.0 are classified as breaking:

- scheduling accounts for total compute capacity; and
- worker threads are shared across all scheduler connections.

Recheck concurrency, fairness, capacity assumptions, and performance tests.

## Operator defaults on OpenShift

The default OpenShift stream labels changed in 3.7.0. Audit label selectors,
LogQL, retention policies, recording rules, and dashboard variables that
assume the previous defaults.

On OCP 4.20, the Operator no longer deploys NetworkPolicies automatically.
This is an operational default change: create required policies explicitly and
verify traffic between all LokiStack components.

## Container working directory

Loki Dockerfiles set the container working directory to `/` in 3.7.0. Derived
images, entrypoints, probes, mounted-file references, and scripts must not
assume the former working directory. Prefer absolute paths.
