# Security and configuration

## Authorization and identity

### Parameterized CIDR authorizers

`CassandraCIDRAuthorizer` retains and applies configured parameters (5.0.3).
Keep parameterized authorizer settings in the configuration; they are no
longer discarded before application.

### Tightened authorization boundaries

Authorization checks around data centers, authorizer handling, and system
keyspaces reject access that earlier behavior could inadvertently allow
(5.0.3). Re-run permission tests after an upgrade instead of carrying forward
assumptions based on accidentally permissive paths.

### Virtual system keyspace grants

Roles can be granted permissions on `system_views` and
`system_virtual_schema` (5.0.5):

```cql
GRANT SELECT ON KEYSPACE system_views TO monitoring_role;
GRANT SELECT ON KEYSPACE system_virtual_schema TO monitoring_role;
```

Use explicit least-privilege grants for monitoring and schema-inspection
clients.

### Identity binding and passwords

A regular user cannot bind an identity to a superuser (5.0.7). Provisioning
that creates this association must run with superuser authority. Password
changes are also rate-limited (5.0.7), so rotation workflows must handle
rejection, wait, and retry rather than issue an unbounded burst.

## Startup and YAML validation

### Audit logging

`audit_logging_options` are sanitized and validated during startup (5.0.3).
Malformed audit configuration is a startup error; validate deployment inputs
before restarting a node.

### Default configuration YAML

Optional settings shipped in the default `cassandra.yaml` remain valid YAML
when uncommented (5.0.4). Configuration generators and uncommenting tools can
use those examples without compensating for previously malformed defaults.

### First boot with disk limits

Setting `data_disk_usage_max_disk_size` before the data directory exists no
longer crashes a node on first boot (5.0.5). It is safe for immutable
configuration to declare the limit before storage initialization.

## Batchlog and guardrail configuration

### Batchlog endpoint strategies

`batchlog_endpoint_strategy` accepts `random_remote`, `prefer_local`,
`dynamic_remote`, and `dynamic` (5.0.3):

```yaml
batchlog_endpoint_strategy: dynamic_remote
```

Choose deliberately according to locality and endpoint-selection needs; do
not reduce the setting to a local/remote boolean.

### Guardrail commands

The final command interface provides `nodetool getguardrailsconfig` and
`nodetool setguardrailsconfig` (5.0.5):

```shell
nodetool getguardrailsconfig
```

Use the commands rather than scripting an earlier provisional interface. A
disk-usage guardrail may also be disabled after its failure threshold has
already tripped (5.0.7); recovery procedures do not have to clear the tripped
state before disabling the guardrail.

## Configuration inventory through `system_views.settings`

Complex values in `system_views.settings` are represented as JSON (5.0.6).
Inventory and monitoring clients must parse those values as JSON rather than
depending on the former representation.

The view redacts security-sensitive settings (5.0.6). Treat redaction as the
supported security boundary and obtain secrets from their authoritative secret
store, not this virtual table.

Configurations that had gone missing from the view are included again for
backward compatibility (5.0.7). Consumers may discover the restored entries,
but should still tolerate additions and redacted values.

## Index-status configuration

`IndexStatusManager` can be forced to use the optimized index-status format
instead of relying on automatic selection (5.0.7). Use the override only when
the deployment requires deterministic format selection, and keep mixed-node
compatibility in mind during rollout.
