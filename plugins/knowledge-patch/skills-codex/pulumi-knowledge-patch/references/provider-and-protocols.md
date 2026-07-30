# Provider and protocol development

This reference consolidates `components-2025`, `3.145.0-3.159.0`,
`3.160.0-3.181.0`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Develop and debug local plugins

`pulumi plugin run` executes a local binary plugin. Debug a source plugin with
the exact form `--attach-debugger plugin=<name>` (`3.160.0-3.181.0`).

Git plugins accept HTTPS URLs, repository subdirectories, and short commit
hashes. An unversioned source resolves to the latest version; `GITHUB_TOKEN` and
`GITLAB_TOKEN` are recognized. Plugin namespaces are inferred and cannot be
overridden in `PulumiPlugin.yaml` (`3.145.0-3.159.0`).

## Version compatibility contracts

Plugins declare their supported CLI range with `requiredPulumiVersion` in
`PulumiPlugin.yaml`; this replaces `pulumiVersionRange`.
`ProviderHandshakeResponse.pulumi_version_range` is removed
(`3.214.1-3.228.0`).

## Configure and diff providers

`Configure` calls receive the provider resource URN and ID, and explicit
providers receive `DiffConfig` for replacement decisions just like default
providers (`3.145.0-3.159.0`). Later host APIs require `Type` on both
`Configure` and `DiffConfig` (`3.229.0-3.248.0`).

Providers can ask the engine to refresh affected resources by default after a
partial failure (`3.160.0-3.181.0`). Provider resources may define
`EnvVarMappings` to rewrite environment variables before the provider sees
them (`3.214.1-3.228.0`).

Node.js provider constructors receive `ignoreChanges`, `replaceOnChanges`,
`customTimeouts`, `retainOnDelete`, and `deletedWith`
(`3.160.0-3.181.0`). Node.js and Python dynamic providers can return inputs
from `read()` to retain diff inputs after refresh (`3.214.1-3.228.0`).

## Invoke and call semantics

Go, Node.js, and Python do not issue provider invokes whose resource
dependencies are unknown. Node.js and Python wait for dependencies embedded in
input properties while retaining compatibility for plain invokes receiving
output arguments (`3.145.0-3.159.0`).

Invokes carry a `preview` flag (`3.199.0-3.214.0`). Provider references passed
as `__self__` on calls are honored, and resource references on the wire include
`Name` and `Type` (`3.214.1-3.228.0`).

Provider methods can return scalars. Go/Python SDK generation and Node.js
program generation support them; Node.js uses `callSingle`
(`3.160.0-3.181.0`). Go program generation emits provider `Call` requests
(`3.214.1-3.228.0`).

## Provider service changes

`StreamInvoke` was removed from the Provider service in 3.161.0; clients and
providers must stop implementing that RPC (`3.160.0-3.181.0`).

The provider protocol and schema now include streaming
`ResourceProvider.List`, exposed by Go's `plugin.Provider`
(`3.229.0-3.248.0`).

Provider and language runtimes support cancellation: Node.js and Python
providers have cancel handlers; Bun, Go, Node.js, and Python wire cancellation
to language-host runs; hosts send `Cancel` when closing plugins
(`3.229.0-3.248.0`).

## Language-host protocol

The language protocol includes `Language.Template` (`3.199.0-3.214.0`) and the
bidirectional-streaming `LanguageRuntime.RunPlugin2` RPC
(`3.229.0-3.248.0`).

Provider handshakes can advertise schema-loader, package-resolver, and mapper
service addresses. Providers launched by the CLI receive the active login in
`PULUMI_API` and `PULUMI_ACCESS_TOKEN` (`3.229.0-3.248.0`).

## Go provider host APIs

Go `plugin.Host` is workspace-stateless. Plugin boot and resolution methods
take `plugin.Context`; plugin-loading functions no longer take `name`. PCL and
schema binding require an explicit schema loader (`3.229.0-3.248.0`).

Go's direct-repository plugin installer supports private GitHub and GitLab
instances (`3.160.0-3.181.0`).

## Schema authoring

Schema names cannot contain whitespace/control characters, conflict with module
paths, or use reserved package names `pulumi` and `input`. Modules below the
index module are binding errors (`3.229.0-3.248.0`).

Schemas may use type-token strings for aliases (`3.145.0-3.159.0`). They also
support extension parameterization, string-enum provider outputs, and function
`multiArgumentInputs`; Go and Python SDK/program generation understand the
latter (`3.229.0-3.248.0`).

SDK generation supports references into parameterized and third-party packages
(`3.214.1-3.228.0`) and extension-parameterized packages
(`3.249.0-3.254.0`).

## State conversion protocol

`ResourceImport` carries parent and properties. Converters can return provider
resources, associate imports with them, receive a schema-loader target, and
request named-ecosystem mappings (`3.249.0-3.254.0`).

## Cross-language component hosts

Python, Go, .NET, and Java source components start runtime-specific provider
hosts; TypeScript and YAML require no distinct entrypoint (`components-2025`).
