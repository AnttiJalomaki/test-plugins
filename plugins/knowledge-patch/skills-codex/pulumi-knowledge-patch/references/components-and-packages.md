# Components and packages

This task-organized reference draws on `components-2025`,
`3.145.0-3.159.0`, `release-notes-117`, `3.160.0-3.181.0`,
`3.182.0-3.198.0`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Author cross-language components

### Source package marker

A component directory becomes a cross-language package when it contains
`PulumiPlugin.yaml`; same-language-only components do not need the file. Pulumi
derives a schema from exposed components and generates consumer SDKs, including
Pulumi YAML SDKs (`components-2025`).

```yaml
runtime: python # nodejs, dotnet, go, java, and yaml are also valid
```

### Expose components by runtime

TypeScript exposes every exported component class and needs no separate entry
point. YAML also needs no entry point. Other runtimes start a component-provider
host (`components-2025`):

- Python uses `_main_.py` and passes classes to
  `pulumi.provider.experimental.component_provider_host(name=..., components=[...])`.
- Go uses the `pulumi-go-provider` v1 builder:
  `infer.NewProviderBuilder().WithNamespace(...).WithComponents(infer.ComponentF(NewComponent)).Build()`,
  then `provider.Run(...)`.
- .NET returns
  `Pulumi.Experimental.Provider.ComponentProviderHost.Serve(args)` from
  `Program.Main`.
- Java starts `com.pulumi.provider.internal.ComponentProviderHost` with the
  package to scan.

Python's experimental component helpers moved from
`pulumi.provider.experimental.provider` to
`pulumi.provider.experimental.component`; the old `provider` namespace now
contains a general provider interface. Python component providers also support
bootstrap-less operation (`3.160.0-3.181.0`).

### Define components in YAML

Pulumi YAML supports top-level `components`. Each component declares inputs,
children, and outputs. Use another authoring language for conditional logic or
map merging (`components-2025`).

```yaml
runtime: yaml
name: my-components
components:
  SecureBucket:
    inputs:
      bucketName:
        type: string
    resources:
      bucket:
        type: aws:s3/bucketV2:BucketV2
        properties:
          bucket: ${bucketName}
    outputs:
      bucketName: ${bucket.id}
```

### Type and initialize components

Python component providers infer enums and resource references.
`@pulumi.type_token` and the static `pulumi_type` class property expose resource
type tokens (`3.160.0-3.181.0`).

Local Node.js components use the version from `package.json` rather than
`0.0.0`; Python component providers can set their version. Node.js component
`initialize` receives options, name, and type (`3.199.0-3.214.0`).

Go and Node.js components send inputs to the engine for diffing and state,
matching Python. Node.js can opt out through
`PULUMI_NODEJS_SKIP_COMPONENT_INPUTS` (`3.199.0-3.214.0`). Node.js schema
inference recognizes enums, `Partial<T>`, and `Required<T>`, and replacement
triggers propagate through remote `Construct` calls (`3.214.1-3.228.0`).

## Add and restore packages

### Git and local sources

`pulumi package add` accepts a Git source, optionally pinned to a tag, or a
relative/absolute component directory. It fetches the source and generates a
local SDK for the project language. Private repository tokens come from the
environment (`components-2025`).

```shell
pulumi package add github.com/myorg/secure-s3-component@v1.0.0
pulumi package add ./components/secure-s3-component
```

Package add records the source under `packages` in `Pulumi.yaml`. Unprefixed
sources are not assumed to be file paths, so use `./` or `../` for local paths
(`3.160.0-3.181.0`). Multiple Git components from one repository can coexist
(`3.199.0-3.214.0`).

Git plugins accept HTTPS URLs, repository subdirectories, and short commit
hashes. An unversioned source resolves its latest version. `GITHUB_TOKEN` and
`GITLAB_TOKEN` are recognized. The namespace is inferred and cannot be
overridden in `PulumiPlugin.yaml` (`3.145.0-3.159.0`).

### Restore declared packages

`pulumi install` processes every `packages` entry in `Pulumi.yaml` and
generates local SDKs. Commit generated SDKs for reproducibility or require
every fresh checkout to run install (`components-2025`). Install now recurses
into local packages (`3.199.0-3.214.0`).

```yaml
packages:
  secure-s3-component: github.com/myorg/secure-s3-component@v1.0.0
resources:
  secureBucket:
    type: secure-s3-component:SecureBucket
```

### Registry resolution

The default package registry source is private, plugin download URLs can be
resolved through the Pulumi Registry, and `pulumi install --file` bypasses the
Registry (`3.160.0-3.181.0`). `pulumi package add` accepts Registry identifiers
(`3.182.0-3.198.0`). Unqualified package names resolve through the Pulumi Cloud
Registry by default (`3.214.1-3.228.0`).

Private packages can be local component dependencies. Package publication
accepts Azure DevOps Git URLs; `pulumi schema check` accepts source packages;
and package add/get-schema install dependencies (`3.214.1-3.228.0`). Package
and plugin flows also accept package references in plugins
(`3.199.0-3.214.0`).

Go's direct-repository installer supports private GitHub and GitLab instances
(`3.160.0-3.181.0`).

## Publish, delete, and bootstrap packages

The package-publishing command introduced experimentally in
`3.145.0-3.159.0` became non-experimental in 3.166.0. The separate
`pulumi template publish` command arrived experimentally in 3.180.0
(`3.160.0-3.181.0`).

`pulumi package delete` removes package versions from the Pulumi Registry
(`3.199.0-3.214.0`). `pulumi package new` bootstraps from a template, and
`pulumi package add --language` works outside a Pulumi project or plugin
(`3.229.0-3.248.0`).

## Schema and SDK generation

### Aliases and functions

Provider schemas may express aliases as type-token strings, not only alias
objects (`3.145.0-3.159.0`).

```json
{ "aliases": ["pkg:index:OldResource"] }
```

`OutputStyleOnly` suppresses a generated plain-function variant
(`3.199.0-3.214.0`). Schemas and Go/Python generation support function
`multiArgumentInputs` (`3.229.0-3.248.0`), while PCL invokes such functions
positionally (`3.249.0-3.254.0`). Provider methods may return scalars; Go and
Python SDK generation and Node.js generation support this, with Node.js using
`callSingle` (`3.160.0-3.181.0`).

### Cross-package references and extensions

SDK generation supports type references into parameterized and third-party
packages (`3.214.1-3.228.0`). Schemas support extension parameterization and
string-enum provider outputs (`3.229.0-3.248.0`). SDK and program generation
also support extension-parameterized packages (`3.249.0-3.254.0`).

### Schema naming constraints

Schema names cannot contain whitespace/control characters or conflict with
module paths; `pulumi` and `input` are reserved package names. Modules nested
beneath the index module are strict binding errors (`3.229.0-3.248.0`).

### CLI version contracts

Projects declare `requiredPulumiVersion` in `Pulumi.yaml`; corresponding checks
are Node.js `requirePulumiVersion`, Python `require_pulumi_version`, Go
`CheckPulumiVersion`, and generated .NET `RequirePulumiVersion`. Plugins use
`requiredPulumiVersion` in `PulumiPlugin.yaml`; the former
`pulumiVersionRange` key was renamed (`3.214.1-3.228.0`).
