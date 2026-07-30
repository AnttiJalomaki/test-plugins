---
name: dotnet-csharp-knowledge-patch
description: .NET and C#
version: null
license: MIT
metadata:
  author: Nevaberry
---


# .NET and C# Knowledge Patch

Use this skill when writing, reviewing, upgrading, or troubleshooting modern
.NET and C# code. Check the project's target framework, SDK selection,
`LangVersion`, runtime identifier, publishing mode, and preview opt-ins before
applying guidance. Preview language and runtime features can require both a
matching compiler/runtime and explicit project configuration.

## Reference index

| Reference | Topics |
|---|---|
| [language.md](references/language.md) | C# extension blocks, properties, lambdas, partial members, assignments, spans, unions, closed hierarchies, control flow, and pointer safety |
| [runtime-and-io.md](references/runtime-and-io.md) | Runtime Async, process control, core types, LINQ, tensors, files, pipes, compression, archives, diagnostics, and platform behavior |
| [serialization-hosting-and-data.md](references/serialization-hosting-and-data.md) | JSON, XML, configuration, validation, hosting, logging, caching, tracing, and EF Core |
| [security-networking-and-interop.md](references/security-networking-and-interop.md) | Certificates, post-quantum cryptography, AES-KWP, X25519, TLS, HTTP, QUIC, native libraries, and COM |
| [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md) | CLI commands, file-based apps, MSBuild, workloads, NuGet, testing, watch/run workflows, and solution filters |
| [deployment-and-ui.md](references/deployment-and-ui.md) | Containers, NativeAOT, native output, runtime configuration, desktop UI, MAUI, and deployment baselines |

## Upgrade triage

Before changing code, identify whether the problem is a compile-time language
change, runtime behavior change, SDK/CLI change, or package/tooling change.

### High-impact runtime and library changes

- `BackgroundService.ExecuteAsync` runs entirely as a `Task`; failures can now
  surface from both `IHost.RunAsync` and `IHost.StopAsync`.
- The runtime no longer installs default termination-signal handlers. Review
  applications that implicitly relied on them.
- `DateOnly.TryParse` and `TimeOnly.TryParse` can throw for invalid input.
  Do not assume every failure is represented only by `false`.
- `CborReader` and `CborWriter` have a default nesting limit.
- ZIP and TAR readers validate CRC32 and header checksums. Treat newly exposed
  failures as corrupt input rather than suppressing validation.
- `ZipArchive.CreateAsync` loads entries eagerly, so entry-read failures can
  occur during archive creation.
- Empty `DeflateStream` and `GZipStream` output now includes framing headers
  and footers.
- `BufferedStream.WriteByte` does not implicitly flush.
- On Unix, `FileStream.IsAsync` and `SafeFileHandle.IsAsync` now report the
  underlying non-blocking state accurately.
- `Environment.TickCount` follows Windows timeout behavior consistently.
- A custom `Type` subclass passed to `Nullable.GetUnderlyingType` now throws.

See [runtime-and-io.md](references/runtime-and-io.md) for exact archive,
process, I/O, diagnostics, and core-library behavior.

### Security and networking changes

- Check `IsSupported` before using `MLKem`, `MLDsa`, or `SlhDsa`; platform
  support depends on the cryptographic backend.
- Composite ML-DSA uses draft-08, and PQC `SecretKey` members were renamed to
  `PrivateKey`.
- Unix requires OpenSSL 1.1.1 or later. Use
  `DOTNET_OPENSSL_VERSION_OVERRIDE` for the OpenSSL override.
- OpenSSL cryptographic primitives are unavailable on macOS, and DSA is no
  longer available there.
- Server-side `SslStream` disables AIA certificate downloads by default.
- Trimmed publications disable HTTP/3 by default.
- Browser HTTP clients stream responses by default.
- `MailAddress` rejects addresses containing consecutive dots.
- `X500DistinguishedName` validation is stricter.

See
[security-networking-and-interop.md](references/security-networking-and-interop.md)
before changing certificate, TLS, HTTP, native-library, or COM code.

### SDK, restore, and deployment changes

- `dotnet new sln` creates SLNX by default.
- CLI informational output and `dotnet watch` logging go to standard error.
- `--interactive` defaults to `true` in user scenarios.
- `dotnet package list` performs restore.
- A versionless `PackageReference` is an error; transitive package auditing is
  enabled during restore.
- Direct references pruned by NuGet raise NU1510, and
  `PrunePackageReference` makes direct prunable references private.
- Workload management defaults to workload-set mode.
- .NET tools are packaged per runtime identifier. Include `any` when a
  portable framework-dependent fallback is required.
- Default container images use Ubuntu. Pin or adapt images that depend on the
  prior distribution's packages, paths, or package manager.
- NativeAOT libraries on Unix use the `lib` prefix.
- File-based apps publish with native AOT by default.

See [sdk-cli-build-and-test.md](references/sdk-cli-build-and-test.md) and
[deployment-and-ui.md](references/deployment-and-ui.md) for migration details.

## C# quick reference

### Extension blocks

C# extension blocks support instance and static members. A named receiver
defines instance members; omit the receiver name for static members.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
        public T this[int index] => source.ElementAt(index);
    }
}
```

Indexers are instance members, so their extension block requires a named
receiver.

### Field-backed properties

Use the contextual `field` token to access a synthesized backing field from an
accessor:

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

If the type already declares an identifier named `field`, use `@field` or
`this.field` for that existing identifier.

### Null-conditional assignment

`?.` and `?[]` can be assignment targets. The right side runs only when the
receiver is non-null:

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

Simple and compound assignments are supported; `++` and `--` are not.

### Preview unions and closed hierarchies

A `union` accepts implicit conversion from each listed case and enables an
exhaustive `switch` when every case is handled:

```csharp
public record Cat(string Name);
public record Dog(string Name);
public union Pet(Cat, Dog);

string Name(Pet pet) => pet switch
{
    Cat cat => cat.Name,
    Dog dog => dog.Name,
};
```

The contextual `closed` modifier restricts direct descendants to the
declaring assembly and enables exhaustive matching. It is not transitive.
Consult [language.md](references/language.md) for preview limitations and the
temporary `ClosedAttribute` requirement.

### Pointer safety is operation-specific

With the preview language version, declarations, address-of, `fixed`,
pointer-targeted `stackalloc`, and `sizeof` on unmanaged types can appear
outside an `unsafe` context. Dereferencing, pointer member or element access,
and function-pointer invocation still require `unsafe`. Pointer-taking runtime
APIs no longer independently carry `RequiresUnsafe`; the pointer operation
still governs the required context.

## New runtime and SDK patterns

### Runtime Async

For a `net11.0` project, enable runtime-managed async suspension with:

```xml
<PropertyGroup>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

It supports NativeAOT and ReadyToRun and exposes the real async call chain in
live stack traces. Use `UseRuntimeAsync=false` to opt out. Do not use the
removed runtime-async environment variables.

### One-shot tool execution

Run a tool without installing it:

```bash
dotnet tool exec --source ./artifacts/package dotnetsay@0.1.0 "Hello"
```

Without `@version`, the latest version is selected. A nearby local tool
manifest can supply the version, and a new download prompts for confirmation.

### Strict JSON

Reject duplicate property names explicitly:

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
```

`JsonSerializerOptions.Strict` also rejects unmapped members, keeps
case-sensitive binding, and enforces nullable annotations and required
constructor parameters. See
[serialization-hosting-and-data.md](references/serialization-hosting-and-data.md)
for generated reference handling, unions, naming policies, and streaming.

### Process execution

Prefer the one-shot `Process.Run*` and `RunAndCaptureText*` APIs for bounded
commands. Use `ReadAllLinesAsync` when output must be streamed as
`ProcessOutputLine` values while preserving the stdout/stderr distinction.
Review platform restrictions before using detached, suspended, inherited
handle, or kill-on-parent-exit behavior.

## Working rules

1. Read the relevant topic reference before relying on a changed default.
2. Verify preview APIs against the actual target framework and SDK.
3. Preserve explicit compatibility choices in project files, environment
   variables, container tags, or serializer options.
4. Re-run restore, build, tests, publish, and deployment checks after an
   upgrade; several failures intentionally move earlier in the workflow.
5. Prefer feature detection such as cryptographic `IsSupported` checks over
   assumptions based only on operating-system names.
