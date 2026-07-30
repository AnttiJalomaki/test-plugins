# Components, packages, and language SDKs

This reference is organized by authoring task. Source batch attribution:
`components-2025`, `3.145.0-3.159.0`, `release-notes-117`,
`3.160.0-3.181.0`, `3.182.0-3.198.0`, `3.199.0-3.214.0`,
`3.214.1-3.228.0`, `3.229.0-3.248.0`, and
`3.249.0-3.254.0`.

## Build a source-backed component package

A component source directory becomes a cross-language package when it contains
`PulumiPlugin.yaml`. Same-language-only components do not need this file.
Pulumi analyzes exported components, derives a schema, and generates SDKs for
consumer languages, including Pulumi YAML.

```yaml
runtime: python # nodejs, dotnet, go, java, or yaml are also supported
```

Exposure differs by runtime:

- TypeScript exposes every exported component class; no separate entry point
  is needed.
- YAML also needs no entry point.
- Python defines `_main_.py` and passes exported classes explicitly to
  `pulumi.provider.experimental.component_provider_host(name=...,
  components=[...])`.
- Go uses the `pulumi-go-provider` v1 builder:
  `infer.NewProviderBuilder().WithNamespace(...).WithComponents(
  infer.ComponentF(NewComponent)).Build()`, then `provider.Run(...)`.
- .NET returns
  `Pulumi.Experimental.Provider.ComponentProviderHost.Serve(args)` from
  `Program.Main`.
- Java starts `com.pulumi.provider.internal.ComponentProviderHost` with the
  package to scan.

For Git plugins, namespaces are inferred and cannot be overridden through
`PulumiPlugin.yaml` as of 3.159.0.

## Author reusable components in YAML

Pulumi YAML supports top-level `components`. A component declares inputs,
child resources, and outputs:

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

Use YAML for direct declarative composition. Conditions, map merging, and
similar logic still require another language. Engine views are enabled by
default from 3.176.0, and Pulumi YAML enables them by default from 3.177.0.
YAML also accepts `object` config types and typechecks component inputs.

## Add packages from source or the Registry

`pulumi package add` accepts:

- a Git repository, optionally pinned to a release tag;
- a relative or absolute component directory;
- a registry package identifier;
- a language selected with `--language`, even outside a Pulumi project or
  plugin.

```shell
pulumi package add github.com/myorg/secure-s3-component@v1.0.0
pulumi package add ./components/secure-s3-component
```

The command fetches or inspects the source and generates a local SDK in the
consumer project's language. Git plugins support HTTPS URLs, subdirectories,
and short commit hashes. An unversioned source resolves its latest version.
`GITHUB_TOKEN` and `GITLAB_TOKEN` are recognized, and private repositories work
when credentials are present.

`pulumi package add` records the source under `packages` in `Pulumi.yaml`.
Unprefixed sources are not treated as file paths; prefix local sources with
`./` or `../`. Unqualified package names now resolve through the Pulumi Cloud
Registry by default.

Component/package workflows also support:

- multiple Git components from one repository;
- private packages as local dependencies;
- Azure DevOps Git URLs for publishing;
- extension-parameterized packages;
- source packages in `pulumi schema check`;
- automatic dependency installation for `package add` and
  `package get-schema`;
- package references inside plugins.

`pulumi package new` bootstraps a package from a template. Conversion from a
third-party source resolves provider plugins through the Pulumi Registry.

## Restore, publish, and remove packages

`pulumi install` processes entries under `packages` in `Pulumi.yaml`, generates
their local SDKs, and recurses into local packages. Generated SDKs may be
committed for reproducibility or ignored, in which case every clean checkout
must run `pulumi install`.

```yaml
packages:
  secure-s3-component: github.com/myorg/secure-s3-component@v1.0.0
resources:
  secureBucket:
    type: secure-s3-component:SecureBucket
    properties:
      bucketName: my-company-secure-assets
outputs:
  bucketName: ${secureBucket.bucketName}
```

The default package Registry source is private. Plugin downloads can resolve
through the Pulumi Registry, while `pulumi install --file` bypasses Registry
resolution. The Go direct-repository installer supports private GitHub and
GitLab instances.

`pulumi package publish` became non-experimental in 3.166.0.
`pulumi package delete` removes a package version from the Pulumi Registry.
`pulumi template publish`, added in 3.180.0, remains experimental.

## Configure component identity and state

- Local Node.js components use the version in `package.json` instead of
  `0.0.0`.
- Python component providers can declare their version.
- Node.js component `initialize` receives resource options, name, and type.
- Go and Node.js components send inputs to the engine for diffing and state
  storage, matching Python. Set `PULUMI_NODEJS_SKIP_COMPONENT_INPUTS` to opt
  out in Node.js.
- Node.js component schema inference understands enums, `Partial<T>`, and
  `Required<T>`.
- Python component providers support resource references, enum inference, and
  enum references.
- Python exposes type tokens with `@pulumi.type_token` and the static
  `pulumi_type` resource-class property.

Autonaming configuration became stable in CLI 3.146.0 and applies to custom
resources, not component resources. It can disable generated names globally or
set behavior per component without program changes.

## Select Node.js, Bun, and TypeScript behavior

The current Node.js SDK requires Node.js 22 or later and supports pnpm 11.
Earlier compatibility moved to a Node.js 20 minimum, added Node.js 24 support,
and changed the compilation target from ES2016 to ES2020. The later test matrix
added Node.js 26 and dropped Node.js 20.

The Node.js SDK can choose Bun as a package manager and configures ESM
automatically unless `--import` or `--require` is supplied. Entrypoint
discovery respects `package.exports`.

Bun is also a distinct native Pulumi runtime through the bundled
`pulumi-language-bun` plugin. It can run programs, plugins, debuggers, and
policy packs; do not confuse this with merely using Bun to install Node.js
packages.

TypeScript 6 is accepted as a peer dependency. Generated TypeScript projects
set both `module` and `moduleResolution` to `nodenext`.

Set the Node.js `production` runtime option in `Pulumi.yaml` to make
`pulumi install` invoke the package manager in production mode and omit
`devDependencies`.

## Select Python behavior

- Dynamic providers work with Poetry and uv.
- Plugin and package discovery understands uv, and `RunPlugin` defaults to a
  virtual environment.
- Lockfile detection selects the project toolchain and reads Poetry and uv
  lockfiles when determining program dependencies.
- Python 3.14 is supported and requires `grpcio>=1.75.1`.
- `pulumi new --generate-only` emits `pyproject.toml` for uv and Poetry
  projects.
- `pulumi.run` supports an async entrypoint that can be awaited natively and
  may return stack outputs.
- `Output.recover` catches and recovers from exceptions during output
  resolution.

Experimental component-provider helpers moved from
`pulumi.provider.experimental.provider` to
`pulumi.provider.experimental.component`; a new general provider interface
uses the former namespace. Component providers also support a bootstrap-less
mode.

## Select Go, .NET, and Java behavior

Generated Go programs and the SDK first moved to Go 1.23, then to Go 1.25. Go
Automation API supports Go 1.26. Go deferred outputs allow creation of an
output handle before the value that later resolves it.

Go's `property.Value` is immutable. Use `property.Path` and
`property.Map.Delete` for structured traversal and removal. `GetPluginInfo`,
`GetPluginPath`, and APIs that create `plugin.Context` accept
`context.Context`.

The Java SDK is generally available at 1.0 with the complete programming model,
cross-language parity, stronger typing, and support for the Java LTS releases
current at launch.

Provider tooling for .NET must migrate code generation imports from
`github.com/pulumi/pulumi/pkg/v3/codegen/dotnet` to
`github.com/pulumi/pulumi-dotnet/pulumi-language-dotnet/v3/codegen`; the old
package is deprecated and scheduled for removal.

## Use Automation API and SDK introspection

- Node.js Automation API adds `previewDestroy` and its generated low-level
  interface exposes `cancel`.
- Python preview exposes its JSON option; command options accept `on_error`
  callbacks for incremental stderr.
- Go, Node.js, and Python inline programs can request run-program behavior for
  refresh and destroy.
- Generated Node.js, Python, and Go Automation APIs expose import.
- Go preview-refresh and refresh support `--import-pending-creates`.

.NET, Go, Java, Node.js, Python, and YAML can look up the project root. Go and
Python expose `pulumiResourceName` and `pulumiResourceType` for a resource's
runtime name and type token.

Go and Python mock monitors expose registered resources for test assertions.
Go tests can inspect the stack export map through `GetCurrentExportMap`.

Node.js and Python plain invokes continue accepting output arguments for
compatibility. Both wait for dependencies embedded in inputs; Go, Node.js, and
Python avoid invokes whose resource dependencies are still unknown.
