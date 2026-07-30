# Clients, observability, and operations

## Logcli

- `logcli` can opt into `ProxyFromEnvironment` in 3.4.0.
- Its output includes common labels in 3.4.0.
- Delete commands are available in 3.6.0.
- Custom request headers can be sent in 3.7.0.

Use environment-derived proxying only when the execution environment's proxy
variables are part of the intended trust and routing configuration.

## Lokitool and ruler checking

- The ruler rule checker can validate a namespace and group in 3.6.0.
- In 3.7.0, `lokitool` adds regex namespace filtering, uses the updated ruler
  path, and accepts alternative TLS environment variables.

## Health and tracing

- Loki adds the `loki health` command in 3.6.0.
- Loki uses OpenTelemetry internally instead of OpenTracing in 3.6.0.
  Existing `JAEGER_`-prefixed environment configuration remains supported,
  and trace export retains Jaeger format.

## Operational UI

In 3.6.0, the Operational UI JavaScript moves into a Grafana plugin while its
server APIs remain within Loki. Enabling the UI in the Helm chart enables the
APIs on queriers, and the gateway forwards UI requests to those APIs.

## Monitoring migration

Meta-monitoring responsibilities move in 3.6.0 from the Grafana
meta-monitoring Helm chart to the Grafana Kubernetes Monitoring Helm chart.
Move ownership and values intentionally so that scraping and dashboards are
not configured twice or dropped during migration.

## Memberlist and IPv6

- Memberlist respects configured interface names when choosing its advertise
  address in 3.4.0.
- In 3.5.0, IPv6 interfaces listed in
  `common.instance_interface_names` are valid advertise-address sources.
- The query frontend can resolve IPv6 addresses in 3.5.0.
- `common.instance_enable_ipv6` propagates to every component in 3.6.0.

Validate both address selection and name resolution when enabling IPv6; the
component-wide enable flag and interface selection solve different parts of
the path.

## Canary behavior

- The canary can run as a Deployment rather than a DaemonSet and can batch log
  pushes in 3.6.0.
- The canary accepts an arbitrary label set for its query in 3.7.0.
- Its readiness probe is configurable in 3.7.0.

Keep the pushed labels and query label set compatible so that the canary tests
the intended route.

## Container execution

Loki Dockerfiles set the working directory to `/` in 3.7.0. Derived images,
entrypoints, and mounted scripts must not rely on the former implicit relative
path.

## Ruler observability

Queries issued by the ruler carry rule name and rule type in their query tags
from 3.4.0. Use those tags in query accounting and troubleshooting.
