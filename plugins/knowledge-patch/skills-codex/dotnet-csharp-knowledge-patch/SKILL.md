---
name: dotnet-csharp-knowledge-patch
description: .NET and C#
version: null
license: MIT
metadata:
  author: Nevaberry
---


# .NET and C# Knowledge Patch

Use this skill when current .NET SDK, runtime, library, deployment, or C# behavior
matters, especially for migrations, changed defaults, and new compiler syntax.

## Start Here

1. Identify the selected SDK, target framework, language version, runtime, and deployment model rather than assuming they move together.
2. For an upgrade, scan **Migration Blockers** before adopting new APIs.
3. Open the matching topic reference for durable details and constraints.
4. Verify platform-conditional behavior on the actual target OS and runtime,
   especially cryptography, TLS, native loading, containers, and UI.
5. In reproducible builds, pin contractual defaults such as the base image,
   workload set, image format, test runner, or JSON policy.

## Reference Index

| Reference | Topics |
| --- | --- |
| [language.md](references/language.md) | C# extension members, properties, spans, lambdas, partial members, assignments, and Visual Basic compatibility |
| [runtime-and-io.md](references/runtime-and-io.md) | Runtime compatibility, core types, I/O, diagnostics, globalization, tensors, and intrinsics |
| [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md) | CLI defaults, tools, restore, workloads, MSBuild, file-based apps, completions, and testing |
| [deployment-and-ui.md](references/deployment-and-ui.md) | Containers, native publishing, base images, file-based deployment, WPF, Windows Forms, and drawing |
| [security-networking-and-interop.md](references/security-networking-and-interop.md) | Certificates, post-quantum crypto, AES-KWP, TLS, HTTP, URI, mail, native loading, and COM |
| [serialization-hosting-and-data.md](references/serialization-hosting-and-data.md) | JSON, XML, hosting, configuration, logging, and EF Core query filters |

## Migration Blockers

### Pin container assumptions

Default container images use Ubuntu. If a Dockerfile, native dependency, or
operations script assumes another distribution's package names, filesystem
layout, or package manager, select a compatible base image explicitly or
adapt the build. Also set `ContainerImageFormat` when Docker versus OCI output
is operationally significant.

See [deployment-and-ui.md](references/deployment-and-ui.md).

### Audit restore and package metadata

Restore audits transitive dependencies. Versionless `PackageReference` items,
invalid package IDs, and some HTTP warnings now fail instead of drifting
through the build. NU1510 identifies direct references that pruning can
remove. Review `deps.json` consumers because packages without runtime assets
are omitted.

See [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md).

### Check CLI automation streams and defaults

User-facing CLI flows default `--interactive` to true. Informational output
unrelated to command results, including `dotnet watch` logging, goes to
standard error. Capture stdout and stderr according to meaning, and pass
noninteractive settings explicitly in automation. New solutions use SLNX by
default, and `dotnet package list` restores before listing.

See [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md).

### Revisit hosting shutdown behavior

The runtime no longer installs default termination-signal handlers, and an
entire `BackgroundService.ExecuteAsync` method runs as a `Task`. Code that
relied on the old synchronous prefix, implicit signal handling, or lifecycle
timing needs explicit validation.

See [runtime-and-io.md](references/runtime-and-io.md) and
[serialization-hosting-and-data.md](references/serialization-hosting-and-data.md).

### Validate serialization contracts

`System.Text.Json` detects property-name collisions. Duplicate JSON names can
be rejected explicitly, and the strict preset also enforces unmapped-member,
case, nullability, and required-constructor rules. `XmlSerializer` includes
obsolete properties instead of silently omitting them. Treat these as schema
changes, not merely parser changes.

See [serialization-hosting-and-data.md](references/serialization-hosting-and-data.md).

### Re-test native loading and interop

Single-file applications no longer probe the executable directory for native
libraries. `DllImportSearchPath.AssemblyDirectory` is now literal: it searches
only the assembly directory. Casting an `IDispatchEx` COM object to `IReflect`
fails. Make native-library locations and COM assumptions explicit.

See [security-networking-and-interop.md](references/security-networking-and-interop.md).

### Review crypto platform requirements

Unix cryptography requires OpenSSL 1.1.1 or later. OpenSSL primitives are not
available on macOS, and post-quantum algorithms require newer platform
providers. Check `IsSupported` before generating or importing post-quantum
keys. Update renamed private-key members and environment variables before
deploying.

See [security-networking-and-interop.md](references/security-networking-and-interop.md).

### Expect stricter validation and exception changes

Stricter checks affect LDAP controls, `X500DistinguishedName`, generic
signature types, JSON names, WPF definitions, and mail addresses with
consecutive dots. Some `System.Drawing` failures now throw
`ExternalException`. Tests should assert the new contract rather than broad
legacy behavior.

## C# Quick Reference

### Extension blocks

An extension block with a named receiver defines instance members. Omitting
the receiver name permits static members. Blocks can add properties, methods,
and operators, so they are broader than classic extension methods.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

### Field-backed properties

Use the contextual `field` token inside an accessor to reach the synthesized
backing field. If the type already has an identifier named `field`, write
`@field` or `this.field` to disambiguate.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

### Null-conditional assignment

`?.` and `?[]` can be assignment targets. The right-hand side runs only for a
non-null receiver. Simple and compound assignment are supported; `++` and
`--` are not.

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

### Spans and overload resolution

Arrays, `Span<T>`, and `ReadOnlySpan<T>` participate in additional implicit
and composed conversions. Generic inference and extension receiver binding
understand spans more directly. Re-run overload-sensitive tests after moving
code to C# 14 because a different viable overload can win.

### Lambda parameter modifiers

Implicitly typed simple lambda parameters may use `scoped`, `ref`, `in`,
`out`, or `ref readonly`. A `params` parameter still needs an explicitly typed
parameter list.

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

### Other language changes

- `nameof(List<>)` returns `"List"` without a type argument.
- Instance constructors and events can be partial. Only the implementing
  constructor supplies `this()` or `base()`; an implementing event supplies
  `add` and `remove`.
- A type can define dedicated compound-assignment operators.

See [language.md](references/language.md) for declaration rules and migration
details.

## API Selection Quick Reference

### Certificates and PFX

Use the algorithm-specific `FindByThumbprint(HashAlgorithmName, ...)` overload
when the thumbprint is not SHA-1. Export PKCS#12 with an explicit preset or
`PbeParameters` so compatibility-oriented 3DES/SHA-1 and modern
AES-256/SHA-256 output are deliberate choices.

### JSON source generation

Generated contexts can preserve references:

```csharp
[JsonSourceGenerationOptions(ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

### Numeric text comparison

`CompareOptions.NumericOrdering` orders embedded digit runs numerically, so
`"2"` sorts before `"10"` and compares equal to `"02"`. Do not use it with
index or prefix operations such as `IndexOf`, `StartsWith`, or `IsPrefix`.

### Tensor slices

Tensor slicing returns a view rather than a copy. Later reads observe changes
to the underlying storage. Arithmetic operators require the element type to
implement the corresponding generic-math interfaces.

## Tooling Quick Reference

- `dotnet tool exec package@version` downloads and runs a tool without
  installing it. It prompts before a new download and honors a nearby local
  manifest version.
- Add the `any` RID with platform RIDs to create a portable,
  framework-dependent tool fallback.
- Pass `--cli-schema` to any CLI command to obtain a JSON command schema.
- Use noun-first `dotnet package ...` and `dotnet reference ...` commands or
  their older aliases.
- Generate native completion scripts with `dotnet completions script`.
- Select `Microsoft.Testing.Platform` under `test.runner` in `global.json`
  when `dotnet test` should use that runner.
- Publishing a file-based application creates a native executable by default;
  add `#:property PublishAot=false` when a dependency is incompatible.

See [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md) for exact
constraints and examples.

## Platform Checks

- On macOS, opt in to client TLS 1.3 through Network.framework only after
  testing protocol availability, buffering, cancellation, zero-byte reads,
  and IDN behavior.
- In trimmed publications, HTTP/3 is disabled by default.
- Browser HTTP clients stream responses by default.
- On x64, `Avx10v2` APIs exist but JIT support remains disabled by default.
- For Windows desktop applications, resolve WPF/Windows Forms type-name
  collisions and treat malformed WPF resources and definitions as failures.

The detailed platform branches are in
[security-networking-and-interop.md](references/security-networking-and-interop.md),
[runtime-and-io.md](references/runtime-and-io.md), and
[deployment-and-ui.md](references/deployment-and-ui.md).
