# Automation and integrations

This reference includes `release-notes-117`, `3.160.0-3.181.0`,
`3.182.0-3.198.0`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Automation API operation controls

Go, Node.js, and Python inline programs can request run-program behavior for
refresh and destroy. Node.js adds `previewDestroy` for a dry-run destroy.
Python preview exposes its JSON option, and Python command options accept
`on_error` callbacks for incremental stderr (`3.182.0-3.198.0`).

Generated low-level Node.js Automation API exposes `cancel`
(`3.214.1-3.228.0`). Generated Node.js, Python, and Go APIs also expose
`import`; Go preview-refresh and refresh can pass
`--import-pending-creates` (`3.249.0-3.254.0`).

Automation API can set bulk JSON configuration with `SetAllConfigJson`
(`3.199.0-3.214.0`). Go preview and update options can carry policy packs
(`3.160.0-3.181.0`). Projects in Automation API settings may omit a runtime
(`3.229.0-3.248.0`).

## Cancellation and output streams

Node.js and Python providers have cancellation handlers. Bun, Go, Node.js, and
Python propagate cancellation to language-host runs, and hosts send `Cancel`
when closing plugins (`3.229.0-3.248.0`).

CLI diagnostics are emitted on stderr, making stdout safer for programmatic
output. `PULUMI_ENABLE_STREAMING_JSON_PREVIEW` controls streaming JSON previews
(`3.182.0-3.198.0`). Engine commands expose structured `--output json`
summaries (`3.229.0-3.248.0`).

## OpenTelemetry traces

`--otel-traces` writes traces to a relative file or sends them to a gRPC
endpoint. Export supports `grpcs://`, authenticated headers, and
`OTEL_RESOURCE_ATTRIBUTES`; provider OpenTracing spans are bridged into
OpenTelemetry (`3.214.1-3.228.0`).

```shell
pulumi preview --otel-traces ./traces.json
```

`TRACEPARENT` attaches CLI spans below an existing trace
(`3.229.0-3.248.0`).

## Pulumi MCP Server

Pulumi's MCP Server exposes CLI and Registry capabilities to MCP-compatible
coding tools, including resource information and infrastructure-management
operations from an editor workflow (`release-notes-117`).

## GitLab pipelines

The GitLab integration supports multiple Pulumi jobs in parallel in one
pipeline on GitLab SaaS and self-managed installations. Authentication through
CI/CD variables avoids personal access tokens (`release-notes-117`).

## Kubernetes Operator

Pulumi Kubernetes Operator 2.0 is generally available with automatic retries
for transient failures, fine-grained refresh control, idempotent updates, and
revised reconciliation and CRD management (`release-notes-117`).

## Neo editor integration

`pulumi neo` runs requested shell and filesystem tools locally while the chat
uses Pulumi Console. Its approval, permission, non-interactive, integration,
and plan controls must be set appropriately for the workspace
(`3.229.0-3.248.0`).

`pulumi neo acp` exposes Neo over Agent Client Protocol stdio with read-only and
plan modes. Resume and failed-update/preview investigation workflows are also
available (`3.249.0-3.254.0`).
