# Runtime, Core Libraries, Diagnostics, and I/O

Use this reference for runtime execution, process and console APIs, core-type
semantics, low-level I/O, compression and archive formats, LINQ, tensors, and
diagnostics. It consolidates relevant material from `10.0-guides`, `10.0`,
`11.0-preview.6-compatibility`, and `11.0-preview.6`.

## Runtime execution and lifecycle

### Runtime Async

Runtime Async is a preview execution model that replaces compiler-generated
async state machines with runtime-managed suspension. It supports NativeAOT
and ReadyToRun and exposes the real async call chain in live stack traces.

A `net11.0` project does not need `EnablePreviewFeatures`. Enable the feature
with:

```xml
<PropertyGroup>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

Set `UseRuntimeAsync=false` to opt out. The former runtime-async environment
variables have been removed.

### Process execution and lifecycle control

`Process.Run*` and `RunAndCaptureText*` provide one-shot execution.
`ReadAllLinesAsync` streams `ProcessOutputLine` values that distinguish stdout
from stderr.

Additional controls include:

- `StartAndForget`;
- `ProcessStartInfo.StartDetached`;
- Windows-only `KillOnParentExit` and `StartSuspended`;
- explicit inherited and standard-stream handles; and
- `SafeProcessHandle` operations for starting, signaling, waiting, opening,
  resuming, and related lifecycle control.

Choose APIs according to platform support and whether output must be captured,
streamed, inherited, or detached.

### Shutdown and host-visible failures

The runtime no longer installs default termination-signal handlers
(`10.0-guides`). Applications that relied on implicit signal handling must
install or configure the behavior they require.

Hosting-specific `BackgroundService` behavior is covered in
[serialization-hosting-and-data.md](serialization-hosting-and-data.md).

## Console and diagnostics

`Console` honors `FORCE_COLOR`. For example,
`FORCE_COLOR=1 dotnet run | tee build.log` retains ANSI color when output is
redirected. Existing `NO_COLOR` support remains.

The default trace-context propagator is the W3C standard
(`10.0-guides`). Review integrations that assumed a different default.

`ActivitySource.CreateActivity` and `ActivitySource.StartActivity` sampling
behavior changed in `10.0-guides`; re-test listeners and samplers that depend
on exact creation or start decisions.

In `10.0`, `ActivitySource` and `Meter` can carry a telemetry schema URL.
Use `ActivitySourceOptions` for the multi-option constructor path.
Out-of-process `Activity` serialization includes events and links.

EventSource trace aggregators can rate-limit root activities per second with a
filter such as:

```text
[AS]*/-ParentRateLimitingSampler(100)
```

## Core types and metadata

### Numeric and temporal behavior

`CompareOptions.NumericOrdering` compares digit sequences numerically:
`"2"` sorts before `"10"`, and `"2"` compares equal to `"02"`. It cannot be
used with index or prefix operations such as `IndexOf`, `StartsWith`, or
`IsPrefix`.

```csharp
int order = CultureInfo.InvariantCulture.CompareInfo.Compare(
    "2", "10", CompareOptions.NumericOrdering);
```

A real `TimeSpan.FromMilliseconds(long)` overload is available and works in
expression trees. The second parameter on the existing two-`long` overload is
no longer optional.

```csharp
Expression<Action> expression =
    () => TimeSpan.FromMilliseconds(1000L);
```

In `11.0-preview.6-compatibility`, `DateOnly.TryParse` and
`TimeOnly.TryParse` can throw for invalid input rather than always returning
`false`. Guard untrusted values accordingly.

The Japanese calendar's minimum supported date was corrected. Validation and
conversion near the lower boundary can therefore change.

`Environment.TickCount` now follows Windows timeout behavior consistently.
Review elapsed-time or timeout logic that depended on the previous
platform-specific behavior.

### Generic math, inline arrays, and reflection

The `10.0-guides` compatibility changes include:

- generic-math shifts behave consistently;
- an explicit struct size is disallowed on an `InlineArray`;
- `FilePatternMatch.Stem` is non-nullable;
- `Type.MakeGenericSignatureType` performs additional argument validation;
- `System.Linq.AsyncEnumerable` is part of the core libraries; and
- reflection and trimming annotations were tightened or removed on several
  APIs, which can expose source or binary incompatibilities.

In `11.0-preview.6-compatibility`,
`Nullable.GetUnderlyingType` throws when passed a custom `Type` subclass
instead of treating every such object as a runtime-provided type.

Some API obsoletions use non-default diagnostic IDs. Suppressions that target
only the usual obsoletion warning may no longer cover them.

### Hardware intrinsics

The x64 intrinsics under `System.Runtime.Intrinsics.X86.Avx10v2` exist in
`10.0`, but JIT support is disabled by default because capable hardware was
not yet available. Do not treat API presence as evidence that the JIT will
emit AVX10.2 instructions.

## LINQ and tensors

`Enumerable`, `Queryable`, and `AsyncEnumerable` provide `FullJoin`.
Their outer joins, plus `Join` and `GroupJoin`, have tuple-returning overloads
that avoid a result selector and can optionally accept an equality comparer
(`11.0-preview.6`).

`System.Numerics.Tensors` is stable rather than experimental in `10.0` and
adds the nongeneric `IReadOnlyTensor`. Slicing returns a non-copying view, so
subsequent access observes the underlying storage. Tensor arithmetic
operators are available only when the element type implements the matching
generic-math interfaces.

## Files, handles, and pipes

`BufferedStream.WriteByte` no longer flushes implicitly. Call `Flush` or
`FlushAsync` when visibility or durability is required before disposal.

On Linux, `DriveInfo.DriveFormat` reports filesystem types.

`SafeFileHandle.Type` identifies the kind of OS object represented.
`CreateAnonymousPipe` creates connected handles whose asynchronous behavior
can be selected independently. `RandomAccess.Read*` and `Write*` work with
non-seekable handles such as pipes (`11.0-preview.6`).

On Unix, these compatibility changes apply:

- `NamedPipeServerStream` with `PipeOptions.CurrentUserOnly` creates its
  socket file with tighter permissions.
- `SafeFileHandle.IsAsync` and `FileStream.IsAsync` accurately report the
  handle's underlying non-blocking state, which can change inspected results.

## Compression

`DeflateEncoder`, `ZLibEncoder`, and `GZipEncoder` provide span-based
compression without a `Stream`. `ZstandardStream` and `ZstandardEncoder` are
available in `System.IO.Compression` (`11.0-preview.6`).

In `11.0-preview.6-compatibility`, `DeflateStream` and `GZipStream` write
headers and footers even for an empty payload. Golden files, content hashes,
or protocols that expect zero bytes for empty compressed content must be
updated.

## ZIP behavior

ZIP entries can be opened with a `FileAccess` mode and expose
`CompressionMethod`.

Two compatibility changes can move failures earlier or expose corrupt data:

- Reading an entry validates its CRC32.
- `ZipArchive.CreateAsync` loads entries eagerly, so entry-reading work and
  related failures happen during archive creation.

## TAR behavior

`TarFile` can explicitly create Pax, Ustar, GNU, or V7 archives, and
`TarReader` accepts GNU sparse format 1.0 (`11.0-preview.6`).

Compatibility details:

- `GnuTarEntry` and `PaxTarEntry` omit `atime` and `ctime` by default
  (`10.0-guides`).
- TAR-reading APIs validate header checksums and reject invalid headers.
- `TarWriter` represents hard-linked files with `HardLink` entries instead of
  emitting them as independent ordinary files
  (`11.0-preview.6-compatibility`).

When archive bytes or entry kinds are part of a stable external contract,
update fixtures and downstream consumers intentionally.
