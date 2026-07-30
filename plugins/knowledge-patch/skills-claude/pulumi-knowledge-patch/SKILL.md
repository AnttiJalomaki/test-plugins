---
name: pulumi-knowledge-patch
description: Pulumi
version: 3.254.0
license: MIT
metadata:
  author: Nevaberry
---


# Pulumi compatibility guidance

Use this skill when maintaining Pulumi projects, providers, components,
Automation API programs, policy packs, or operational tooling. It emphasizes
removed surfaces, changed defaults, lifecycle semantics, and newer workflows
that are easy to miss when upgrading.

## How to use this skill

1. Read `requiredPulumiVersion` from `Pulumi.yaml` when present, then inspect
   language manifests, lockfiles, provider versions, and backend type.
2. Check breaking changes and removed commands before changing an established
   workflow.
3. Use the task-specific reference for exact CLI flags, resource options,
   runtime behavior, package workflows, or provider contracts.
4. Treat experimental commands and APIs as unstable unless the reference says
   they graduated.
5. Exercise previews, state backups, and focused tests before applying
   lifecycle or state changes.

## Reference index

| Reference | Topics |
| --- | --- |
| [CLI operations and configuration](references/cli-operations-and-configuration.md) | Operation flags, environment controls, config, output, logging, authentication, templates, and plugin execution |
| [State, import, and resource lifecycle](references/state-import-and-lifecycle.md) | State repair, imports, protection, targeting, hooks, replacement, secrets, and backend migration |
| [Components, packages, and language SDKs](references/components-packages-and-runtimes.md) | Cross-language components, package lifecycle, YAML, runtimes, Automation API, and SDK changes |
| [Providers, schemas, PCL, and protocols](references/providers-schemas-pcl-and-protocols.md) | Provider contracts, schema authoring, code generation, PCL, conversion, invokes, and protocol changes |
| [Cloud, policy, and operations](references/cloud-policy-and-operations.md) | Pulumi Cloud API, ESC, CI, Kubernetes Operator, policy, Neo, direct-resource operations, and DIY backends |

## Breaking changes first

### Remove retired CLI and deployment-settings workflows

`pulumi query` is gone. Local `Pulumi.<stack>.deploy.yaml` files and their
`deployment settings init`, `pull`, `configure`, `env`, and
`push`/`update`/`up` commands are also gone. Manage deployment settings in
Pulumi Cloud:

```shell
pulumi deployment settings get
pulumi deployment settings edit
pulumi deployment settings destroy
```

The associated `--config-file` flag and SDK file-reading helpers are no longer
available.

### Upgrade language runtimes deliberately

The Node.js SDK requires Node.js 22 or later and supports pnpm 11. Current
generated TypeScript projects use `nodenext` for both `module` and
`moduleResolution`, and TypeScript 6 is accepted as a peer dependency.
Generated Go programs and SDK modules target Go 1.25; Automation API supports
Go 1.26.

When a Node.js project's `Pulumi.yaml` sets the `production` runtime option,
`pulumi install` omits `devDependencies`.

### Update provider protocol implementations

`StreamInvoke` was removed from the Provider service. The old
`ProviderHandshakeResponse.pulumi_version_range` field is also removed, and
`PulumiPlugin.yaml` uses `requiredPulumiVersion` rather than
`pulumiVersionRange`.

Provider implementations must account for newer required fields and services:

- `Configure` and `DiffConfig` require the provider type.
- Plugin loading functions no longer take a separate name.
- Schema and PCL binding require an explicit schema loader.
- Provider hosts use `plugin.Context` and no longer retain workspace state.
- The provider protocol includes streaming `ResourceProvider.List`.

### Treat logout as destructive configuration cleanup

`pulumi logout` now removes all configuration for the selected backend,
including shared temporary agent credentials and the current tokenless
backend. Do not use it as a harmless token-only reset.

### Recheck automatic log handling

Encrypted automatic CLI logging is enabled by default. Logs live under
`~/.pulumi/logs`; property-value secrets are redacted, and `pulumi logs ls`,
`decrypt`, `rm`, and `share` manage captures. Adjust retention procedures
rather than relying on the earlier opt-in environment variable.

### Preserve current YAML config semantics

The temporary behavior that preserved scalar types from YAML configuration was
reverted. The CLI warns when YAML `null` would become an empty string. Use
explicit config types and test consumers rather than depending on the reverted
behavior.

### Respect stricter schema and PCL contracts

Provider schema names cannot contain whitespace or control characters, collide
with module paths, use reserved package names `pulumi` or `input`, or define
modules nested beneath the index module. Labels on PCL `package` blocks are
deprecated, and component inputs are typechecked.

### Observe corrected failure and import behavior

Failed resource registrations produce faulted outputs rather than unknown
outputs. Diffs nested inside `Output` values are no longer ignored. Importing
with a noncanonical identifier no longer schedules a later deletion, so remove
workarounds built around the former behavior.

## High-value CLI workflows

### Produce machine-readable operation summaries

Use `--output json` with `up`, `preview`, `refresh`, `destroy`, and `import`.
For the first four operations, affected-resource records include URN, type,
name, operation, and parent.

```shell
pulumi preview --output json
```

Non-UTF-8 strings appear as `b"<base64>"` in diffs and JSON rather than
invalid text.

### Run program code when state operations need it

`refresh` and `destroy` skip program execution unless `--run-program` is set.
The flag is useful for short-lived credentials, dynamic providers, and other
program-supplied operation context.

```shell
pulumi refresh --run-program
pulumi destroy --run-program
PULUMI_RUN_PROGRAM=true pulumi up --refresh
```

`preview` and `up` accept the same setting when combined with `--refresh`.

### Control one operation without editing stack configuration

`up`, `preview`, `destroy`, and `refresh` accept `--override-env` for imported
environment substitutions and `--skip-config-validation` to bypass the
project's config schema. Treat the latter as an exceptional recovery tool.

Every CLI flag also has a generic `PULUMI_OPTION_*` form; for example,
`--refresh` maps to `PULUMI_OPTION_REFRESH`.

### Call Pulumi Cloud directly

`pulumi api <operation-or-path>` is the canonical direct Cloud API client. It
supports fields, headers, request bodies, path templates, content negotiation,
and dry runs. Use `list` and `describe` to inspect the API, and `--paginate`
to combine cursor pages into one JSON envelope.

### Bootstrap without a runtime

Projects may omit a runtime. `pulumi project new -y` creates a minimal project,
`pulumi new` aliases `pulumi project new`, and the CLI can run through `npx`:

```shell
npx pulumi preview
```

## High-value lifecycle controls

### Express replacement relationships

Use `replaceWith` to replace a resource when another resource is replaced,
including transitive or mutual replacement groups. Use `replacement_trigger`
when an arbitrary value change should force replacement.

```typescript
const app = new aws.ec2.Instance("app", args, {
  replaceWith: [database],
  replacementTrigger: rolloutVersion,
});
```

Consult the lifecycle reference for language availability and remote-component
propagation.

### Recover failed operations with hooks

Resource hooks are available in Go, Node.js, and Python. `OnError` hooks can
implement retries; all provider errors reach error hooks, and a successful PCL
hook command retries the failed operation. A failing after-hook fails the
deployment. Destroy operations with delete hooks must run the program.

### Repair state with focused commands

`pulumi state taint` and `untaint` control replacement on the next update.
`pulumi state protect` changes protection directly in state. Bulk deletion can
accept several URNs and order them by dependency; `--all` removes every state
entry and requires exceptional care.

### Migrate stacks between backends

`pulumi stack migrate` imports a stack from another backend into the current
one and re-encrypts configuration secrets and state under the destination
secrets provider. Back up and verify both sides before migration.

## High-value component and package workflows

### Publish source-backed components across languages

A component directory with `PulumiPlugin.yaml` can be analyzed into a schema
and consumer SDKs. The file is unnecessary for same-language-only components.
TypeScript and YAML expose components directly; Python, Go, .NET, and Java run
a component provider host.

### Add and restore packages

`pulumi package add` accepts registry identifiers, Git sources, and explicit
local paths such as `./component`. It records sources under `packages` in
`Pulumi.yaml`; `pulumi install` restores those packages and recurses into local
ones.

Unqualified package names resolve through the Pulumi Cloud Registry by
default. Use `pulumi install --file` when registry resolution must be bypassed.

### Use YAML components for declarative composition

Pulumi YAML supports top-level `components` containing typed inputs, child
resources, and outputs. Use another authoring language when the definition
needs conditional logic or map merging.

## Verification targets

After an upgrade, prioritize tests that:

- preview imports, replacements, hooks, and protection changes against copied
  state;
- verify Node.js, Python, Go, Java, and Bun runtime/toolchain selection;
- exercise package restoration from registry, Git, and local sources;
- assert secret propagation through invokes, hooks, stack references, and
  imported state;
- validate generated schemas and code against strict module and type rules;
- parse JSON operation summaries, stderr diagnostics, and non-UTF-8 values;
- confirm backend tags, automatic-log retention, and logout behavior;
- compare Automation API flags with their CLI equivalents.
