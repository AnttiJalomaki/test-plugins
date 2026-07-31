# Runtime, Core Libraries, Diagnostics, and I/O

This topic reference incorporates compatibility guidance from
`10.0-guides` and API guidance from `10.0`.

## Buffered I/O

`BufferedStream.WriteByte` no longer causes an implicit flush. Code that needs
bytes to reach the underlying stream at a protocol boundary, durability point,
or before handing off the stream must call `Flush` or `FlushAsync` explicitly.
Do not use individual byte writes as a flushing mechanism.

## Trace Context and Activity Sampling

The default trace-context propagator is W3C. Verify cross-service headers and
trace-parent expectations when interoperating with code that assumed another
default.

`ActivitySource.CreateActivity` and `StartActivity` have changed sampling
behavior. Keep listener and sampler tests explicit about whether an activity
is created, started, and recorded instead of relying on older incidental
behavior.

`ActivitySource` and `Meter` can carry a telemetry schema URL.
`ActivitySourceOptions` is the constructor path when several source options
must be configured together. Out-of-process `Activity` serialization includes
events and links, so downstream schemas and payload sizes should account for
them.

EventSource trace aggregators support rate limiting of root activities. A
filter such as the following caps roots at 100 per second:

```text
[AS]*/-ParentRateLimitingSampler(100)
```

Choose a limit intentionally so load protection does not silently discard the
diagnostic volume needed for incident analysis.

## Filesystem and Archive Metadata

On Linux, `DriveInfo.DriveFormat` reports filesystem types. Callers that
previously treated the value as absent or generic should accept real platform
filesystem names.

`GnuTarEntry` and `PaxTarEntry` omit `atime` and `ctime` by default. Set those
fields when archive consumers require them; do not assume merely creating an
entry preserves access and change timestamps.

## Shutdown and Protocol Validation

The runtime no longer installs default termination-signal handlers.
Applications that require graceful cleanup or a particular signal-to-exit
mapping must register and coordinate that behavior explicitly.

LDAP `DirectoryControl` parsing is stricter. Malformed or nonconforming data
that previously passed may now fail, so validate externally supplied controls
and update tests that depended on permissive parsing.

## Core Type and Metadata Compatibility

- Generic-math shift operations now behave consistently. Re-test custom
  numeric types that compensated for inconsistent shifts.
- A struct with `InlineArray` cannot specify an explicit size. Remove the
  competing layout declaration rather than trying to make both contracts
  authoritative.
- `FilePatternMatch.Stem` is non-nullable. Update nullable annotations and
  eliminate branches that treated a valid match's stem as null.
- `Type.MakeGenericSignatureType` performs additional argument validation.
  Invalid signature construction now fails earlier.
- `System.Linq.AsyncEnumerable` is part of the core libraries. Check for
  duplicate type or extension exposure from compatibility packages.
- Reflection and trimming annotations were tightened or removed on several
  APIs. Re-run trim analysis and check binary compatibility; old suppression
  assumptions may no longer match the API metadata.

## Numeric String Ordering

`CompareOptions.NumericOrdering` compares embedded digit sequences as numbers:

```csharp
int order = CultureInfo.InvariantCulture.CompareInfo.Compare(
    "2", "10", CompareOptions.NumericOrdering);
```

`"2"` sorts before `"10"`, while `"2"` and `"02"` compare equal. Numeric
ordering is not valid for index or prefix operations such as `IndexOf`,
`StartsWith`, or `IsPrefix`. Use it for comparison and sorting only.

## `TimeSpan.FromMilliseconds`

There is a real `TimeSpan.FromMilliseconds(long)` overload, so a long-valued
call is representable in an expression tree:

```csharp
Expression<Action> expression = () => TimeSpan.FromMilliseconds(1000L);
```

The existing overload taking two `long` values no longer makes its second
parameter optional. Supply both arguments when that overload is intended.

## Tensor Contracts and Views

`System.Numerics.Tensors` is stable rather than experimental and includes the
nongeneric `IReadOnlyTensor` contract. Slicing a tensor returns a non-copying
view. Access through the slice observes later changes to the underlying
storage, so copy explicitly when isolation is required.

Tensor arithmetic operators are constrained by generic math: the element type
must implement the interface corresponding to the requested operator. A
numeric-looking type without that interface does not gain the operator merely
by being stored in a tensor.

## AVX10.2 Intrinsics

The x64 intrinsic API surface is under
`System.Runtime.Intrinsics.X86.Avx10v2`. JIT support remains disabled by
default because capable hardware was not available when the API shipped.
Presence of the API is therefore not evidence that a production code path can
execute it; retain runtime feature checks and a fallback implementation.
