---
name: dotnet-csharp-knowledge-patch
description: .NET and C#
version: null
license: MIT
metadata:
  author: Nevaberry
---


# .NET and C# Knowledge Patch

Use this skill when writing, upgrading, reviewing, or troubleshooting current
.NET and C# code. Start with the breaking-change checks below, then read the
reference file that matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [language.md](references/language.md) | C# extension members, properties, spans, lambdas, partial members, assignments, preview unions, closed hierarchies, indexers, labeled jumps, and pointer rules |
| [runtime-io.md](references/runtime-io.md) | Runtime behavior, core types, processes, files, pipes, compression, archives, LINQ, tensors, NativeAOT, and hardware requirements |
| [security-networking.md](references/security-networking.md) | Certificates, post-quantum cryptography, AES-KWP, X25519, TLS, HTTP, QUIC, OpenSSL, and networking compatibility |
| [serialization-data-diagnostics.md](references/serialization-data-diagnostics.md) | JSON, XML, CBOR, configuration binding, validation, EF Core filters, tracing, metrics, and sampling |
| [sdk-cli-packaging.md](references/sdk-cli-packaging.md) | CLI behavior, tools, file-based apps, solutions, testing, containers, workloads, NuGet, MSBuild, and templates |
| [hosting-platform-interop.md](references/hosting-platform-interop.md) | Hosting lifecycle, environment variables, native loading, COM, Windows desktop, MAUI, and platform-specific behavior |

## Upgrade checks: breaking behavior first

### Audit .NET 10 migrations

- Rebuild container assumptions: default images use Ubuntu, so verify package
  names, package-manager commands, filesystem paths, and native dependencies.
- Treat informational CLI output as stderr. Keep stdout parsers restricted to
  command results, and capture both streams when retaining diagnostics.
- Expect `dotnet new sln` to create SLNX, `dotnet package list` to restore, and
  local tool installation to create a missing manifest.
- Review restore failures and warnings: transitive auditing, required package
  versions, pruning diagnostics, invalid IDs, and HTTP-source errors are
  stricter.
- Check single-file native loading. The executable directory is no longer an
  implicit probe location, and `AssemblyDirectory` means only the assembly
  directory.
- Re-test JSON and XML contracts. JSON property-name collisions are rejected,
  while obsolete XML-serializable properties are no longer silently excluded.
- Review hosted services: the complete `BackgroundService.ExecuteAsync` method
  runs as a task, including its synchronous prefix.
- Pin or adapt behavior affected by changed HTTP/3, browser streaming, URI,
  email-address, logging, configuration-null, cryptography, and desktop rules.

### Audit .NET 11 preview migrations

- Expect malformed or unusual archives to fail earlier: ZIP CRC32 and TAR
  header checks are enforced, and async ZIP creation loads entries eagerly.
- Do not assume `DateOnly.TryParse` or `TimeOnly.TryParse` is exception-free for
  invalid input; guard untrusted parsing where necessary.
- Compare byte fixtures for empty deflate and gzip payloads because framing is
  now emitted even with no content.
- Revisit host shutdown handling. Failed background services propagate from
  `RunAsync` and `StopAsync`.
- Supply missing certificate-chain material on servers; server-side AIA
  downloading is disabled by default.
- Revalidate Unix deployment artifacts, file and pipe behavior, and NativeAOT
  library names, including the `lib` prefix.
- Revalidate the deployment CPU floor after the JIT hardware-baseline change.
- Replace macOS DSA use and require Android API level 24 or later for MAUI.
- Stop relying on transitive `Newtonsoft.Json` from VSTest or inferred Mono
  launch targets for .NET Framework applications.
- Update warning suppression and CI parsing for specific obsoletion IDs,
  NU1703, and NU5052.

## C# quick reference

### C# 14 extension blocks

A named receiver enables instance members. Omitting the receiver name enables
static members.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

Extension indexers are also available in the later preview and require a named
receiver because an indexer is an instance member.

### Field-backed properties

Use contextual `field` inside a property accessor to access its synthesized
backing field. If the declaring type already has an identifier named `field`,
use `@field` or `this.field` for that existing identifier.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

### Assignment and lambda changes

- `?.` and `?[]` can be assignment targets for simple and compound assignment.
  The right side runs only when the receiver is non-null; `++` and `--` are not
  supported through null-conditional access.
- Types can define dedicated compound-assignment operators.
- Implicitly typed lambda parameters may carry `scoped`, `ref`, `in`, `out`, or
  `ref readonly`; `params` still requires explicit parameter types.
- Partial instance constructors and events require one defining and one
  implementing declaration. Constructor initializers belong only on the
  implementing declaration.

### Span and overload review

C# 14 adds first-class conversions among arrays, `Span<T>`, and
`ReadOnlySpan<T>`. These conversions participate in extension receivers,
composed conversions, generic inference, and overload resolution. Recompile
and test overload-sensitive code because a span-aware candidate may now win.

### Preview language features

- A leading `with(...)` collection-expression element supplies constructor or
  factory arguments such as capacity or an equality comparer.
- `union` declares an exhaustively matchable value over listed case types.
- `closed` restricts direct descendants to the declaring assembly; the
  restriction is not transitive.
- `break label;` and `continue label;` target a labeled enclosing loop or
  switch.
- Pointer declarations and several pointer-producing operations no longer need
  an `unsafe` context under the preview language version, but dereference,
  pointer access, and function-pointer invocation still do.

Read [language.md](references/language.md) before adopting preview syntax; it
records required support types and the features that remain unfinished.

## Runtime and library quick reference

### Process execution

Use the `Process.Run*` and `RunAndCaptureText*` families for one-shot process
execution. `ReadAllLinesAsync` streams `ProcessOutputLine` values that retain
the stdout/stderr distinction. Review detached, suspended, inherited-handle,
kill-on-parent-exit, signal, wait, and resume controls before replacing custom
process wrappers.

### Runtime Async

For a `net11.0` project, opt into the preview runtime-managed async execution
model with:

```xml
<PropertyGroup>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

Use `UseRuntimeAsync=false` to opt out. Do not use the removed environment
variables. Read [runtime-io.md](references/runtime-io.md) for NativeAOT,
ReadyToRun, and stack-trace implications.

### JSON safety

Reject duplicate property names explicitly when consuming untrusted JSON:

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
```

`JsonSerializerOptions.Strict` also rejects unmapped members, preserves
case-sensitive binding, and enforces nullable annotations and required
constructor parameters. Source-generated contexts can select
`JsonKnownReferenceHandler.Preserve` for cyclic graphs.

### Cryptography feature detection

Check `IsSupported` before using `MLKem`, `MLDsa`, or `SlhDsa`; platform
support depends on the installed cryptographic provider. Treat experimental
diagnostics as API stability signals, not as runtime feature detection. For
macOS TLS 1.3 client opt-in and its protocol and behavior tradeoffs, read
[security-networking.md](references/security-networking.md).

## CLI and SDK quick reference

### File-based applications

Publishing a file-based app produces a native executable by default. Disable
native AOT for incompatible dependencies and add project references through
file directives:

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
```

Use `#:include` to compose source files or reference prebuilt DLLs. Matching
duplicate SDK, property, and package directives are allowed across includes.

### Tools and automation

- `dotnet tool exec package@version` downloads and runs a tool without
  installing it; pin the version for reproducible automation.
- `--cli-schema` emits a command's machine-readable JSON shape.
- `dotnet completions script` generates native completion scripts.
- Noun-first `package` and `reference` commands coexist with older forms.
- `dotnet sln` can create and modify `.slnf` solution filters.
- Use `dotnet run -e KEY=VALUE` for runtime environment variables.

### Test runner selection

Select Microsoft.Testing.Platform in `global.json` or with
`DOTNET_TEST_RUNNER`. Its newer controls cover dependency building, current
runtime use, module exclusions, cancellation, live output, and device
selection. Keep VSTest dependency assumptions separate from runner selection.

### Container output

Console projects can invoke `PublishContainer` directly. Set
`ContainerImageFormat` to `Docker` or `OCI` when consumers require a stable
format, and consult [sdk-cli-packaging.md](references/sdk-cli-packaging.md)
before changing multi-architecture or Podman workflows.

## Working rule

Treat project manifests, source, tests, deployed platforms, and observed
behavior as authoritative. Apply preview guidance only when the project opts
into the corresponding target framework or language version.
