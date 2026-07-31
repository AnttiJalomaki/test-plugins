# Migrations and Compatibility

## Compatibility policy

### Apply Semantic Versioning expectations (since 1.0.0)

k6 follows Semantic Versioning. Breaking changes are limited to major
releases and receive deprecation warnings in advance. Each major version gets
critical fixes for at least two years. The supported public API surface is
explicitly identified for extension and integration authors.

Treat experimental outputs, preview libraries, and experimental modules as
outside the same stability expectations; they can carry breaking changes in a
minor release.

## Removed execution control

### Replace externally controlled execution in v2 (since 2.0.0)

The `externally-controlled` executor and the `k6 pause`, `k6 resume`,
`k6 scale`, and `k6 status` commands were removed without direct replacements.
A script configured with that executor does not start.

Choose an executor that represents the workload instead, such as:

- `ramping-vus` for a changing number of virtual users.
- `constant-vus` for fixed virtual-user concurrency.
- `constant-arrival-rate` for a fixed iteration arrival rate.

## HTTP call compatibility

### Repair extra GET and HEAD arguments (since 1.8.0)

`http.get()` and `http.head()` warn when extra positional arguments are passed.
The values remain ignored, but the warning identifies a call that does not
match the supported method signature. Move intended request settings into the
correct parameters object or remove the extra values.

## Migration map

Use the topic references for the remaining compatibility work:

| Area | Required review |
| --- | --- |
| Cloud CLI and script options | [Cloud command migration](cli-cloud-and-configuration.md#cloud-command-migration) |
| Config file and HTTP API defaults | [Configuration paths and services](cli-cloud-and-configuration.md#configuration-paths-and-services) |
| Extension resolution and Go imports | [Extensions and dependencies](extensions-and-dependencies.md) |
| Summary, Rate, and OpenTelemetry changes | [Outputs and observability](outputs-and-observability.md) |
| FID replacement and browser traffic | [Browser testing](browser-testing.md#browser-metrics-and-cloud-diagnostics) |
| Stable modules and WebSockets | [Scripting, security, and protocols](scripting-security-and-protocols.md) |

## Version-aware review procedure

1. Compare the developer, CI, container, Cloud, and archive k6 versions.
2. Identify deprecated interfaces before changing the major version.
3. Inspect environment variables as well as script options; several removed
   switches never appear in JavaScript.
4. Rebuild Go extensions against the v2 module path and supported compiler.
5. Exercise CLI automation while asserting process exit status, not only
   output text.
6. Check output schemas and metric labels consumed by dashboards or alerting.
7. Run browser tests with redirects, frames, and request waits when those paths
   are present.
