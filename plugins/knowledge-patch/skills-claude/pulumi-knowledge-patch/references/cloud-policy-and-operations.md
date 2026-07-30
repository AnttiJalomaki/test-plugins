# Cloud, policy, and operations

This reference is organized by platform task. Source batch attribution:
`release-notes-117`, `3.145.0-3.159.0`, `3.160.0-3.181.0`,
`3.182.0-3.198.0`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Call Pulumi Cloud APIs

`pulumi api <operation-or-path>` calls any Pulumi Cloud API operation or raw
path. It handles fields, headers, input, body content, path templates, content
negotiation, and dry runs.

Use `pulumi api list` and `pulumi api describe` to inspect the OpenAPI surface.
`--paginate` combines cursor pages into a single JSON envelope, and
`--emit-events` sends pagination progress to stderr. Use `--output`, not the
earlier `--format` spelling.

Pulumi's MCP Server exposes CLI and Registry capabilities through MCP,
including resource discovery and infrastructure-management operations.

## Operate with Neo

`pulumi neo` is available without `PULUMI_EXPERIMENTAL`. It runs
agent-requested shell and filesystem tools locally in the working directory,
while Pulumi Console backs the chat.

Operational modes include:

- `--print` for non-interactive prompts;
- approval and permission controls;
- `--disable-integrations`;
- a plan mode that must be chosen before the first message and blocks file
  writes, `pulumi up`, and pull-request creation until approval;
- `pulumi neo acp` for Agent Client Protocol operation over standard I/O, with
  read-only and plan session options;
- `pulumi neo resume` to restore chat history;
- `--debug-update` and `--debug-preview` to investigate failed operations.

```shell
pulumi neo --print '<prompt>'
```

## Operate resources directly

`pulumi do` supports direct resource operations. Earlier `create`, `patch`, and
`delete` behavior required `--stateless` to opt into direct-provider execution.
Current `create` and `delete` can operate statefully, and `upsert` statefully
creates or updates a resource.

Number and boolean inputs may be expressions. `--provider` can reuse provider
configuration from provider state. Project-scoped PCL input receives the
selected stack's organization and short stack name.

## Manage ESC credentials and workflows

ESC environments can rotate long-lived credentials, including AWS IAM access
keys, on a schedule or on demand. Rotation uses two secrets for smooth consumer
transitions, supports separate administrator and consumer roles, and can call
webhooks after rotation.

The Pulumi ESC GitHub Action injects environment configuration and secrets into
a workflow and can run ESC commands. Pair it with `pulumi/auth-actions` for
OIDC-based tokenless Pulumi Cloud authentication.

`pulumi env open-request` now submits the change request for approval rather
than retaining a draft. Use `--reason` to provide context to approvers.

## Integrate CI and Kubernetes

The GitLab integration supports several Pulumi jobs in parallel within one
pipeline on GitLab SaaS and self-managed installations. Its CI/CD-variable
authentication avoids personal access tokens.

Pulumi Kubernetes Operator 2.0 is generally available. It adds automatic retry
for temporary failures, fine-grained refresh controls, idempotent updates, and
reworked reconciliation and CRD management.

## Author and run policy packs

The experimental Go Policy as Code SDK supports policy authoring, and Go
Automation API preview and update options can include policy packs. Policy code
can access a Pulumi `Context`.

`pulumi policy install` installs policy-pack dependencies. Policy operations
also install missing policy analyzer plugins automatically.

The engine supports policy-violation severity overrides, and the CLI displays
each violation's severity.

`pulumi policy analyze` evaluates a policy pack against existing state. With
`--file <stack-export>`, it analyzes an exported state file without requiring a
selected stack or backend login. Local policy packs can resolve ESC
environments.

`pulumi policy new` accepts `--runtime-options`. Policy, policy-group, and
related list commands accept `--output <format>`.

## Apply policy to discovered resources

Pulumi Insights can apply existing Policy as Code rules to discovered
resources, including infrastructure not managed by Pulumi IaC. Link Insights
Accounts to Policy Groups alongside stacks to cover discovered AWS, Azure,
OCI, and Kubernetes resources.

## Manage templates and Registry content

`pulumi new` consumes templates from Pulumi Cloud and accepts qualified
Registry template names. It lists templates published with
`pulumi template publish`.

Private Registry template publication and resolution no longer require
`PULUMI_EXPERIMENTAL`. Set `PULUMI_DISABLE_REGISTRY_RESOLVE=true` when
`pulumi new` must avoid Registry lookup.

`pulumi package publish` is stable; `pulumi template publish` remains
experimental. `pulumi package delete` removes a package version from the
Registry.

## Operate DIY backends

S3, Azure Blob, Google Cloud Storage, PostgreSQL, and local DIY backends support
stack-tag create, read, update, delete, automatic system tags, and tag-filtered
stack listings.

```shell
pulumi stack tag set environment production
pulumi stack ls --tag-filter environment=production
```

Tags are versioned JSON in `.pulumi-tags` files next to checkpoints. Existing
untagged stacks remain valid, and deleting a stack removes its tag file.

DIY backends support zstd state compression. The legacy non-project operating
mode is deprecated. `pulumi stack rm --remove-backups` also removes DIY
backups.

## Recover and observe backend operations

Engine journaling is enabled by default. Set
`PULUMI_DISABLE_JOURNALING=true` only when deliberately disabling it.

```shell
PULUMI_DISABLE_JOURNALING=true pulumi up
```

The service backend automatically repairs snapshot-integrity faults and emits
an error event so the repair can be diagnosed. Imported service-backed secrets
managers are reconfigured for the target stack when required.

Automatic encrypted CLI logs and OpenTelemetry export are covered in the CLI
reference.
