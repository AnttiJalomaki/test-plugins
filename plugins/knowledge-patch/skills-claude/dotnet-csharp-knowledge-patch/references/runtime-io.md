# Runtime, I/O, and Core Libraries

## Runtime and I/O compatibility (`10.0-guides`)

### Buffering, filesystems, and TAR metadata

- `BufferedStream.WriteByte` no longer flushes implicitly. Flush explicitly at
  the required visibility or durability boundary.
- On Linux, `DriveInfo.DriveFormat` reports filesystem types.
- `GnuTarEntry` and `PaxTarEntry` omit `atime` and `ctime` by default. Set the
  timestamps deliberately if an archive consumer requires them.

### Trace propagation and activity sampling

The default trace-context propagator is the W3C standard. The sampling behavior
of `ActivitySource.CreateActivity` and `ActivitySource.StartActivity` also
changed. Test code that assumes a particular propagation format or that an
activity is created under a specific listener/sampler arrangement.

### Core types and metadata

- Generic-math shift operations behave consistently with the updated rules.
- An explicit struct size is disallowed on a type with `InlineArray`.
- `FilePatternMatch.Stem` is non-nullable.
- `Type.MakeGenericSignatureType` validates its arguments more thoroughly.
- `System.Linq.AsyncEnumerable` is part of the core libraries.
- Reflection and trimming annotations were tightened or removed on several
  APIs. This can expose both source and binary incompatibilities; recompile and
  rerun trimming analysis rather than assuming annotation-only effects.

## Core-library APIs (`10.0`)

### Numeric string ordering

`CompareOptions.NumericOrdering` compares embedded digit sequences by numeric
value. Consequently, `"2"` sorts before `"10"`, and `"2"` compares equal to
`"02"`. Do not combine this option with index or prefix operations such as
`IndexOf`, `StartsWith`, or `IsPrefix`.

```csharp
int order = CultureInfo.InvariantCulture.CompareInfo.Compare(
    "2", "10", CompareOptions.NumericOrdering);
```

### `TimeSpan.FromMilliseconds` overloads

A real `TimeSpan.FromMilliseconds(long)` overload now works in expression
trees. The second parameter of the existing two-`long` overload is no longer
optional.

```csharp
Expression<Action> expression = () => TimeSpan.FromMilliseconds(1000L);
```

### Tensor contracts and slice views

`System.Numerics.Tensors` is stable and includes the nongeneric
`IReadOnlyTensor` interface. Slicing returns a non-copying view; later reads
observe changes in the underlying storage. Tensor arithmetic operators are
available only when the element type implements the matching generic-math
interfaces.

### AVX10.2 intrinsics

The x64 intrinsics are exposed under
`System.Runtime.Intrinsics.X86.Avx10v2`, but JIT support is disabled by default
because capable hardware was not available when the support shipped. Do not
assume that the presence of the API means generated AVX10.2 instructions are
enabled.
