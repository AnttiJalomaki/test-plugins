# Providers, schemas, PCL, and protocols

This reference is organized by provider-development task. Source batch
attribution: `3.145.0-3.159.0`, `3.160.0-3.181.0`,
`3.199.0-3.214.0`, `3.214.1-3.228.0`, `3.229.0-3.248.0`, and
`3.249.0-3.254.0`.

## Implement provider lifecycle calls

Provider `Configure` calls receive the provider resource's URN and ID.
Explicit providers receive `DiffConfig` for replacement decisions just like
default providers. Current `Configure` and `DiffConfig` requests require the
provider type.

The engine correctly inherits the `provider` option during resource
registration without carrying a default provider across package boundaries.
Provider resources can define `EnvVarMappings` to remap environment variables
before the provider receives them.

Provider protocols can request default refreshes of affected resources after a
partial failure. Node.js and Python providers have cancel handlers, while Bun,
Go, Node.js, and Python wire cancellation through language-host executions.
Hosts send `Cancel` when closing plugins.

## Implement calls, invokes, and returns

Provider methods can return a scalar rather than an object. Go and Python SDK
generation support scalar returns; Node.js program generation uses the
`callSingle` SDK path.

Invokes receive a `preview` flag. If an invoke has a secret input but the
provider lacks secret support, the engine marks its outputs secret.

Go, Node.js, and Python skip provider invokes while resource dependencies are
unknown. Node.js and Python also wait for dependencies present in input
properties, while plain invokes continue accepting output arguments.

Go program generation can emit provider `Call` requests. The engine populates
`Name` and `Type` in wire-level `ResourceReference` values and honors a
provider reference passed to a call through `__self__`.

## Update removed and added protocol surfaces

`StreamInvoke` was removed from the Provider service in 3.161.0. Provider
implementations and clients must remove that RPC.

Newer protocol surfaces include:

- server-streaming `ResourceProvider.List`, exposed through Go's
  `plugin.Provider`;
- bidirectional-streaming `LanguageRuntime.RunPlugin2`;
- `Language.Template`;
- provider handshakes that return schema-loader, package-resolver, and mapper
  service addresses;
- provider invokes that identify preview calls.

CLI-launched providers receive the active login via `PULUMI_API` and
`PULUMI_ACCESS_TOKEN`.

## Update host and plugin contracts

Go `plugin.Host` is workspace-stateless. Plugin boot and resolution functions
take `plugin.Context`, plugin-loading functions no longer accept a separate
name, and PCL/schema binding requires an explicit schema loader.

Projects can declare `requiredPulumiVersion` in `Pulumi.yaml`. The corresponding
language checks are:

- Node.js `requirePulumiVersion`;
- Python `require_pulumi_version`;
- Go `CheckPulumiVersion`;
- generated .NET `RequirePulumiVersion`.

Plugins also declare supported CLI ranges with `requiredPulumiVersion` in
`PulumiPlugin.yaml`. This replaces `pulumiVersionRange`; the
`ProviderHandshakeResponse.pulumi_version_range` field has been removed.

## Propagate provider and analyzer options

Node.js provider constructors receive `ignoreChanges`, `replaceOnChanges`,
`customTimeouts`, `retainOnDelete`, and `deletedWith` rather than dropping
them.

Go `AnalyzerResourceOptions` includes `Parent`, and resource transforms receive
the parent URN. Imports supplied by transforms are honored. Replacement
triggers pass through remote-component `Construct` calls.

`customTimeouts` includes a `read` field for resource read timeouts.

## Author valid provider schemas

Provider schemas can express aliases as type-token strings:

```json
{
  "aliases": ["pkg:index:OldResource"]
}
```

Schema identifiers have strict constraints:

- names cannot contain whitespace or control characters;
- names cannot conflict with module paths;
- `pulumi` and `input` are reserved package names;
- a module nested under the index module is a bind error.

Schemas support extension parameterization and string-enum provider outputs.
SDK and program generation support extension-parameterized packages and type
references into parameterized or third-party packages.

Functions may declare `multiArgumentInputs`; Go and Python SDK/program
generation supports them. PCL invokes these functions positionally.

`OutputStyleOnly` suppresses generation of a function's plain return variant
and emits only its output-style form.

## Generate and convert programs

`pulumi convert --from=<plugin>@<version>` pins a converter plugin. Conversion
can bridge Terraform providers automatically, and PCL generation recognizes
`try` and `can`. Parameterized package/provider resources are supported by
imports.

```shell
pulumi convert --from=terraform@1.2.3
```

Provider plugins for third-party conversion resolve through the Pulumi
Registry. The HCL language runtime and converter are downloaded on demand, but
converting a Terraform program to the `hcl` target is rejected.

Program and SDK generation supports:

- scalar provider method returns;
- resource options for imports;
- extension-parameterized packages;
- asset, archive, provider, and resource-reference values in imports;
- provider calls in generated Go;
- multi-argument function inputs.

## Author PCL programs

PCL supports:

- resource ranges and maps of ranged resources indexed by key;
- `read` blocks that query a resource by ID without registration;
- parameterized providers;
- config values that must be read as secrets;
- resource hooks, including `onError`;
- state snippets retained as PCL blocks;
- integer types for integer literals, list/tuple indices, and the `element` and
  `range` builtins.

PCL applies resource-schema defaults. Invoke-derived config defaults resolve to
the invoke result, and schema-declared nested output fields are populated so
optional objects can be traversed safely. Component inputs are typechecked.

Labels on `package` blocks are deprecated.

Hook functions receive resource options; a failing after-hook fails the
deployment. A successful generated `onError` hook command retries the failed
operation. Engine deployment options can target state snippets by UUID with
`TargetSnippets`.

## Work with state converters

`ResourceImport` includes parent and properties fields for hierarchy and
property filtering. State converters can:

- return explicit provider resources and associate resources with them;
- receive a schema-loader target;
- request ecosystem mappings by name rather than only for their own
  ecosystem;
- supply inputs and outputs for direct-state imports;
- operate with parameterized and extension-parameterized providers.

Import generation preserves assets, archives, and resource references in
nested rich values.
