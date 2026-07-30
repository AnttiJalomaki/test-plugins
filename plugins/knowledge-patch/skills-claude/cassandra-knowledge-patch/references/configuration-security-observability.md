# Configuration, Security, and Observability

## Configuration loading and validation

### Parameterized CIDR authorization

`CassandraCIDRAuthorizer` retains and applies configured parameters (since
5.0.3). Deployments can parameterize the authorizer without those values being
discarded while the configuration is applied.

### Audit logging startup validation

`audit_logging_options` are sanitized and validated during startup (since
5.0.3). A malformed audit configuration should be handled as a startup
configuration error rather than something that will remain latent until audit
logging is exercised.

### Default YAML

Optional entries in the distributed `cassandra.yaml` remain valid YAML when
uncommented (since 5.0.4). This also matters to downstream configuration
templating and parsing tools that start from the default file.

### Unified Compaction sizes

Unified Compaction validates its minimum and target size settings correctly
(since 5.0.5). Invalid size combinations are rejected; do not rely on an
invalid combination being accepted and normalized later.

### Initial disk-usage configuration

Configuring `data_disk_usage_max_disk_size` before the data directory exists
does not crash a node on first boot (since 5.0.5). It is safe to provision the
limit as part of an initial configuration rather than deferring it until after
the directory has been created.

### Table-name length

DDL rejects table names that would lead to filenames that are too long (since
5.0.6). Identifier-generating automation should surface this validation error
and choose a shorter name instead of expecting a later filesystem failure.

## Authorization and identity

### Corrected authorization boundaries

Authorization checks around data-center handling, authorizers, and system
keyspaces are stricter (since 5.0.3). Requests previously accepted because of
overly broad access can be rejected after an upgrade. Re-test roles that
operate on system resources or depend on data-center-scoped behavior.

### Virtual system keyspaces

Permissions can be granted on `system_views` and
`system_virtual_schema` (since 5.0.5):

```cql
GRANT SELECT ON KEYSPACE system_views TO monitoring_role;
GRANT SELECT ON KEYSPACE system_virtual_schema TO monitoring_role;
```

Use explicit grants for monitoring roles rather than assuming these virtual
system keyspaces cannot participate in normal authorization.

### Identity binding

A regular user cannot bind an identity to a superuser (since 5.0.7).
Provisioning must perform such an association with authority appropriate to
the target superuser; a workflow driven by an ordinary user is rejected.

### Password-change rate limiting

Password changes are rate-limited (since 5.0.7). Provisioning and rotation
clients must tolerate rejection or delay when they submit rapid repeated
changes; they should not assume every immediate rotation attempt is accepted.

## Guardrail administration

### Configuration commands

Guardrail configuration is exposed by the finalized, simplified
`nodetool getguardrailsconfig` and `setguardrailsconfig` interface (since
5.0.5):

```shell
nodetool getguardrailsconfig
```

Use `setguardrailsconfig` with the intended setting arguments. Automation
should target this final command interface rather than an earlier draft shape.

### Disabling a tripped disk guardrail

The disk-usage guardrail can be disabled after its failure threshold has
already been reached (since 5.0.7). A tripped state no longer prevents the
administrative disable operation.

## Virtual settings inventory

### JSON values

Complex values in `system_views.settings` are represented as JSON (since
5.0.6). Consumers must parse those fields as JSON rather than depending on the
older representation.

### Sensitive-value redaction

Security-sensitive values in `system_views.settings` are redacted (since
5.0.6). Monitoring and configuration-inventory systems must treat redaction as
expected and must not use the view as a source from which to reconstruct
secrets.

### Configuration coverage

Settings that had been missing from `system_views.settings` are present again
for backward compatibility (since 5.0.7). Inventory consumers can discover
those configurations through the virtual table, while still applying the JSON
and redaction rules above.

## JMX management surfaces

### Prepared-statement invalidation

`StorageService.dropPreparedStatements` is exposed over JMX (since 5.0.6).
Management clients can invalidate prepared statements through the
`StorageService` interface.

### Native connection limit

`StorageProxyMBean` exposes
`NativeTransportMaxConcurrentConnectionsPerIp` (since 5.0.6). JMX monitoring
can read the configured per-IP native transport connection cap directly.

### Bootstrap visibility

The `StorageService` JMX MBean is available while a node is bootstrapping
(since 5.0.5). Bootstrap-aware automation does not need to wait for the node to
finish bootstrap before using that MBean.

## Diagnostics and operational expectations

### SAI table statistics

`nodetool tablestats` includes selected SAI index state and query-performance
metrics (since 5.0.3):

```shell
nodetool tablestats
```

Use the existing table-statistics command when diagnosing SAI state rather
than assuming index detail requires a separate command.

### Failure-detector timing

The default maximum interval used by the failure detector is calculated
correctly (since 5.0.7). Clusters that leave the value at its default may see
different failure-detection timing after upgrading, even without an explicit
configuration change.

### Handled exceptions

Exceptions that Cassandra catches and handles no longer generate heap dumps
(since 5.0.7). Incident automation should not require a dump artifact as proof
that such an exception occurred.

### Direct-memory reporting

`nodetool gcstats` reports direct-memory usage correctly (since 5.0.7).
Monitoring baselines built from earlier incorrect values should be recalibrated
before interpreting a post-upgrade change as a memory regression.
