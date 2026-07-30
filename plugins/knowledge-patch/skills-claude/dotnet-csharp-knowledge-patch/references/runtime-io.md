# Runtime, I/O, compression, and core libraries

## Runtime execution and shutdown

### Runtime Async

.NET 11 Preview 6 introduces Runtime Async (`11.0-preview.6`), a preview
execution model in which the runtime manages suspension rather than using
compiler-generated async state machines. It supports NativeAOT and ReadyToRun
and exposes the real asynchronous call chain in live stack traces.

A `net11.0` project does not need `EnablePreviewFeatures`. Opt in with:

```xml
<PropertyGroup>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

Use `UseRuntimeAsync=false` to opt out. Previous runtime-async environment
variables have been removed.

### Termination behavior

The runtime no longer installs default termination-signal handlers in .NET 10
(`10.0-guides`). Applications that depended on those handlers must own their
signal behavior.

### Tick count and calendar boundaries

`Environment.TickCount` follows Windows timeout behavior consistently in .NET
11. Review code that depended on earlier platform-specific behavior.

The Japanese calendar's minimum supported date has been corrected. Re-test
validation and conversion near the lower boundary.

## Process execution and console output

The `Process.Run*`, `RunAndCaptureText*`, and `ReadAllLinesAsync` APIs provide
one-shot process execution or a stream of `ProcessOutputLine` values that
distinguish stdout from stderr.

Lifecycle and handle controls include:

- `StartAndForget` and `ProcessStartInfo.StartDetached`.
- Windows-only `KillOnParentExit` and `StartSuspended`.
- Explicit inherited handles and explicit standard-stream handles.
- `SafeProcessHandle` operations to start, signal, wait, open, and resume.

`Console` honors `FORCE_COLOR`. For example,
`FORCE_COLOR=1 dotnet run | tee build.log` retains ANSI color in redirected
output. Existing `NO_COLOR` behavior remains available.

## Buffered, random-access, and asynchronous I/O

`BufferedStream.WriteByte` no longer implicitly flushes in .NET 10. Call
`Flush` or `FlushAsync` where durability or immediate visibility matters.

In .NET 11 Preview 6:

- `SafeFileHandle.Type` identifies the represented operating-system object.
- `CreateAnonymousPipe` creates connected handles whose asynchronous behavior
  can be selected independently.
- `RandomAccess.Read*` and `Write*` accept non-seekable handles such as pipes.
- On Unix, `SafeFileHandle.IsAsync` and `FileStream.IsAsync` accurately report
  the underlying non-blocking state; observed property values can therefore
  change after upgrading.
- On Unix, `NamedPipeServerStream` with `PipeOptions.CurrentUserOnly` creates
  its socket file with tighter permissions.

## Compression and archives

### Compatibility checks

.NET 11 Preview 6 hardens archive handling
(`11.0-preview.6-compatibility`):

- Reading a ZIP entry validates its CRC32 and rejects corrupt or mismatched
  data.
- TAR readers validate header checksums and reject invalid headers.
- `ZipArchive.CreateAsync` loads entries eagerly, so I/O and parsing failures
  can move into archive creation.
- `TarWriter` records hard-linked files as `HardLink` entries instead of
  independent ordinary files.
- `DeflateStream` and `GZipStream` emit headers and footers for an empty
  payload, changing zero-length golden bytes.

In .NET 10, `GnuTarEntry` and `PaxTarEntry` omit `atime` and `ctime` by default.

### New buffer and format APIs

Span-based `DeflateEncoder`, `ZLibEncoder`, and `GZipEncoder` compress buffers
without requiring a `Stream`. `ZstandardStream` and `ZstandardEncoder` add
Zstandard support through `System.IO.Compression`.

ZIP entries can be opened with a `FileAccess` mode and expose their
`CompressionMethod`. `TarFile` can explicitly create Pax, Ustar, GNU, or V7
archives, and `TarReader` accepts GNU sparse format 1.0.

## Core types and parsing

### Type and metadata validation

.NET 10 core-library compatibility changes include:

- Generic-math shift operations behave consistently.
- `InlineArray` cannot be combined with an explicit struct size.
- `FilePatternMatch.Stem` is non-nullable.
- `Type.MakeGenericSignatureType` performs additional argument validation.
- `System.Linq.AsyncEnumerable` is part of the core libraries.
- Reflection and trimming annotations were tightened or removed on several
  APIs, which can expose source or binary incompatibilities.

In .NET 11 Preview 6, `Nullable.GetUnderlyingType` throws when passed a custom
`Type` subclass rather than treating it like a runtime-provided type. Some
obsoletions also use non-default diagnostic IDs; update targeted suppressions.

### Date-only and time-only parsing

`DateOnly.TryParse` and `TimeOnly.TryParse` can throw for invalid input in .NET
11 instead of always returning `false`. Guard untrusted parsing where an
exception-free boundary is required.

### Numeric string ordering

`CompareOptions.NumericOrdering` compares embedded digit sequences
numerically, so `"2"` sorts before `"10"` and `"2"` compares equal to `"02"`.
It cannot be combined with index and prefix operations such as `IndexOf`,
`StartsWith`, or `IsPrefix`.

```csharp
int order = CultureInfo.InvariantCulture.CompareInfo.Compare(
    "2", "10", CompareOptions.NumericOrdering);
```

### TimeSpan expression trees

`.NET 10` provides a real `TimeSpan.FromMilliseconds(long)` overload, so it
works in expression trees. The existing two-`long` overload's second parameter
is no longer optional.

```csharp
Expression<Action> expression =
    () => TimeSpan.FromMilliseconds(1000L);
```

## LINQ and tensors

`Enumerable`, `Queryable`, and `AsyncEnumerable` add `FullJoin`. Their outer
joins, `Join`, and `GroupJoin` also offer tuple-returning overloads that avoid
a result selector and optionally accept an equality comparer.

`System.Numerics.Tensors` is stable in .NET 10 and includes nongeneric
`IReadOnlyTensor`. Slicing returns a non-copying view, so reads made later
observe underlying-storage changes. Tensor arithmetic operators are available
only when the element type implements the corresponding generic-math
interfaces.

## NativeAOT and hardware

NativeAOT native-library outputs on Unix use the `lib` prefix in .NET 11
Preview 6. Update artifact discovery, packaging, and deployment paths.

The .NET 11 JIT has a new minimum hardware baseline. Revalidate every
deployment target rather than assuming an older CPU remains supported.

The x64 AVX10.2 intrinsics are exposed under
`System.Runtime.Intrinsics.X86.Avx10v2` in .NET 10, but JIT support remains
disabled by default because capable hardware was not yet available.
