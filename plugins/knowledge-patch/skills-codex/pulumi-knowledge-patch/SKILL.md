---
name: pulumi-knowledge-patch
description: Pulumi
version: 3.254.0
license: MIT
metadata:
  author: Nevaberry
---


# Pulumi

Use this skill when creating, operating, testing, or extending Pulumi projects,
providers, components, packages, policy packs, or Automation API programs. Read
the topic reference that matches the work before choosing a CLI flag, runtime
baseline, state-repair operation, package workflow, or provider protocol.

Prefer project manifests, generated SDKs, provider schemas, state, and observed
CLI behavior when they disagree with general guidance. Treat state mutation,
secret display, backend migration, logout, and direct-resource operations as
high-impact actions: inspect the selected backend and stack first, preview when
possible, and obtain confirmation before changing shared infrastructure.

## Reference index

| Reference | Topics |
| --- | --- |
| [CLI and operations](references/cli-and-operations.md) | Engine commands, flags, environment variables, machine output, logging, Neo, `pulumi api`, and `pulumi do` |
| [State, imports, and backends](references/state-imports-and-backends.md) | State repair, import semantics, DIY backends, journaling, stack migration, and stack references |
| [Components and packages](references/components-and-packages.md) | Cross-language components, YAML components, package installation, Registry workflows, schema generation, and code generation |
| [Resources and engine](references/resources-and-engine.md) | Resource options, replacement, hooks, providers, secrets, views, diffs, and lifecycle behavior |
| [Languages and runtimes](references/languages-and-runtimes.md) | Node.js, TypeScript, Python, Go, Java, .NET, Bun, testing, and runtime entrypoints |
| [PCL and conversion](references/pcl-and-conversion.md) | PCL syntax and typing, HCL, conversion, snippets, reads, and generation |
| [Provider and protocol development](references/provider-and-protocols.md) | Provider services, host APIs, handshakes, schema contracts, invokes, and language-host protocol |
| [Policy, security, and authentication](references/policy-security-and-auth.md) | Policy analysis, OIDC, ESC, secrets, authentication, and Insights |
| [Automation and integrations](references/automation-and-integrations.md) | Automation API, Kubernetes Operator, GitLab, tracing, and editor integrations |

## Breaking changes and removed surfaces

### Use the current Node.js baseline

The Node.js SDK requires Node.js 22 or later. New TypeScript projects use
`nodenext` for both `module` and `moduleResolution`, TypeScript 6 is accepted,
and pnpm 11 is supported. Do not rely on the earlier Node.js 20 minimum.

### Remove obsolete CLI and deployment-settings usage

`pulumi query` is removed. Local `Pulumi.<stack>.deploy.yaml` deployment
settings and their `deployment settings init`, `pull`, `configure`, `env`, and
`push`/`update`/`up` commands are also removed, as are `--config-file` and the
SDK helpers that read those files. Manage deployment settings in Pulumi Cloud:

```shell
pulumi deployment settings get
pulumi deployment settings edit
pulumi deployment settings destroy
```

### Update provider protocol implementations

The Provider service no longer exposes `StreamInvoke`. The old
`ProviderHandshakeResponse.pulumi_version_range` field is gone, and
`PulumiPlugin.yaml` uses `requiredPulumiVersion` rather than
`pulumiVersionRange`. Provider tooling should use the .NET code generator from
`pulumi-dotnet`; the former package under the core Pulumi repository is
deprecated and scheduled for removal.

### Expect logout to clear backend configuration

Logout deletes all backend configuration, shared temporary agent credentials,
and the current tokenless backend. Inspect `credentials.json` and active login
context before scripting logout.

### Treat automatic encrypted logs as enabled

The CLI captures encrypted logs for every command by default. Property-value
secrets are redacted. Use `pulumi logs ls`, `decrypt`, `share`, and `rm` to
manage captures; the older opt-in environment variable is no longer required.

## High-value CLI controls

### Produce structured engine results

`up`, `preview`, `refresh`, `destroy`, and `import` accept `--output json`.
Summaries for the first four include each affected resource's URN, type, name,
operation, and parent.

```shell
pulumi preview --output json
```

Use `--urns` when human-readable displays need complete URNs. Diagnostics and
pagination progress go to stderr, which keeps stdout suitable for machine data.
Non-UTF-8 strings appear as `b"<base64>"` rather than invalid text.

### Supply options through the environment

Any command flag can use `PULUMI_OPTION_*`; uppercase the flag name and replace
hyphens with underscores. Dedicated controls include `PULUMI_STACK`,
`PULUMI_PARALLEL`, `PULUMI_PARALLEL_DIFF`, and `PULUMI_RUN_PROGRAM`.

```shell
PULUMI_STACK=dev PULUMI_OPTION_REFRESH=true pulumi preview
```

### Run program code for refresh or destroy when required

Use `--run-program` when the program establishes short-lived credentials,
loads operation context, or defines dynamic providers needed by refresh or
destroy. Preview and update accept it with `--refresh`.

```shell
pulumi refresh --run-program
pulumi destroy --run-program
pulumi up --refresh --run-program
```

### Scope operations explicitly

Target-aware operations accept `--exclude <URN>` and
`--exclude-dependents`. `up`, `preview`, `destroy`, and `refresh` also accept
`--override-env` and `--skip-config-validation` for one-operation overrides.

Never use `--show-secrets` in captured CI logs unless plaintext output is
explicitly required and the log sink is trusted.

## Resource lifecycle quick reference

### Coordinate replacement

Use `replaceWith` to replace a resource whenever another referenced resource
is replaced. Relationships are transitive and may be mutual. C# and YAML do
not yet expose this option. Use `replacement_trigger` when an arbitrary value,
rather than another resource replacement, should force replacement.

```typescript
const app = new aws.ec2.Instance("app", {}, {
  replaceWith: [database],
  replacementTrigger: deploymentFingerprint,
});
```

### Repair state deliberately

`pulumi state taint` schedules replacement; `untaint` cancels it. State delete
accepts multiple URNs and dependency-orders their removal, while `--all`
removes every resource entry. Use `pulumi state protect` to set protection
directly in state. Export and secure a recovery copy before destructive repair.

### Use hooks with program execution in mind

Go, Node.js, and Python support resource lifecycle hooks and `OnError` retry
hooks. Destroy operations involving delete hooks must run the program. A
failing after-hook fails the deployment; a successful error-hook command
retries the failed operation. Hooks receive resource type, name, options, and
secret-preserving values where supported.

### Adopt and update in one deployment

An `import` resource option can adopt an existing object and update it in the
same deployment; its import ID remains through updates. Resource transforms
may supply the import option. Import files can include providers, assets,
archives, resource references, inputs, and outputs; supplying outputs imports
state directly without a provider read.

## Components and packages quick reference

### Expose source components cross-language

A source component directory needs `PulumiPlugin.yaml` for cross-language use;
same-language-only components do not. TypeScript and YAML expose components
without a separate provider-host entrypoint. Python, Go, .NET, and Java must
start their runtime-specific component provider host.

### Install declared package sources

`pulumi package add` accepts Registry identifiers, Git sources, and explicit
local paths such as `./component`. It records sources in `Pulumi.yaml`.
`pulumi install` restores declared packages, recurses into local packages, and
generates local SDKs. Unqualified names resolve through the Pulumi Cloud
Registry; use `pulumi install --file` when registry resolution must be bypassed.

```shell
pulumi package add github.com/acme/component@v1.0.0
pulumi package add ./components/component
pulumi install
```

### Declare compatible CLI versions

Projects can set `requiredPulumiVersion` in `Pulumi.yaml`; language SDKs expose
matching runtime checks. Plugins use the same key in `PulumiPlugin.yaml` for
their supported CLI range.

## Authentication and secret handling

### Prefer short-lived OIDC login in CI

`pulumi login --oidc-token` exchanges a raw token or `file://` token for a
short-lived Pulumi Cloud credential. The organization must trust the issuer and
authorize the token claims and audience. The CLI can infer organization, team,
and user claims; use `--oidc-org`, `--oidc-team`, or `--oidc-user` to scope the
identity explicitly and `--oidc-expiration` to change the default two-hour
lifetime.

### Preserve secret taint

Invokes with secret inputs produce secret outputs even when the provider lacks
secret support. Node.js and Python resource hooks receive secrets as `Output`
values. Undecryptable stack-reference outputs are omitted, while a missing
Python stack-reference output returns a missing value rather than throwing.

## Direct and automated workflows

### Use stateful `pulumi do` operations

`pulumi do` supports stateful `create`, `delete`, and `upsert`; number and
boolean inputs may be expressions. The older `--stateless` mode retains direct
provider behavior. Verify the selected project, stack, and provider state
before invoking direct-resource changes.

### Bootstrap runtime-free projects

Projects and Automation API settings may omit a runtime. `pulumi project new
-y` creates a minimal project, `pulumi new` is its alias, and the CLI can run
through `npx pulumi`.

### Call Pulumi Cloud APIs directly

`pulumi api <operation-or-path>` exposes the Cloud API surface. Use `list` or
`describe` to inspect it, `--paginate` to combine cursor pages, and `--output`
for the final representation. Use dry-run handling before mutating API calls.
