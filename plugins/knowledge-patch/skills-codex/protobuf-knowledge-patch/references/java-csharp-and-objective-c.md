# Java, C#, and Objective-C

Use this reference for generated Java layout and JSON behavior, C# package and
Edition changes, and Objective-C runtime or descriptor migrations.

## Java generated layout in Edition 2024

The `edition-2024-announcement` replaces `java_multiple_files` with the
`nest_in_file_class` feature. Edition 2024 generates classes in their own files
by default.

The default outer class name is always the camel-cased proto filename with
`Proto` appended. For example, `foo/bar_baz.proto` becomes `BarBazProto`. Set
`java_outer_classname` when this generated identifier is not suitable.

## Java large enums

The Edition 2024 `large_enum` feature permits generated Java enum-like types
beyond the language's usual enum-constant limit. These types emulate enums but
do not support every normal enum operation, including `switch`.

`34.0` adds `large_enum` support to the Java lite runtime and correctly handles
large enums with aliased values. Recompile both full and lite consumers after
enabling the feature; do not assume source compatibility with code that used
language `enum` operations.

## Java initialization and recursion

`34.0` deprecates generated `isInitialized()` accessors on message types that
have no required fields. Calls on those types can newly trigger deprecation
diagnostics; remove redundant checks while retaining them where required-field
semantics still apply.

The same release extends recursion-limit enforcement to Java JSON containing
`Any` nested inside `Any`. Deep payloads that previously parsed can fail once
the configured limit is reached.

## Java timestamp and JSON behavior

`36.0-rc1` changes `Timestamps.parse()` to attempt strict, non-lenient parsing
first. If that fails, it warns before accepting an input through lenient
parsing. Treat the warning as input-normalization or validation work rather
than assuming the value is clean.

When `JsonFormat` is asked to sort map keys, it now uses natural Java `String`
comparison instead of UTF-8 `ByteString` comparison. Serialized key order can
change for non-ASCII or otherwise differently ordered keys. Update golden
files and signatures that intentionally depend on sorted output.

`JsonFormat` also gains an opt-in strict JSON parser. Enable it for callers
that require strict input validation; do not assume the default parser becomes
strict automatically.

## Java removed and future APIs

The experimental Java `FieldOrder` enum is removed in `36.0-rc1`. Remove
references rather than depending on the experimental surface.

Java now warns about possible `OneofDescriptor` naming collisions ahead of Q1
2027 breaking changes. Resolve names and regenerate before that release rather
than suppressing the warning.

The compiler also warns on the deprecated `java_generic_service` option in
`36.0-rc1`. Use the RPC framework's generator plugin.

## C# validation and packages

Since `30.0-migration`, C# reports UTF-8 enforcement failures earlier when a
protobuf `string` contains invalid UTF-8. Ensure parse and assignment paths
surface the error at their new point rather than relying on later
serialization to fail.

`34.0` extends recursion-limit enforcement to C# JSON well-known types
containing deeply nested arrays. Treat limit failures as parse errors for
untrusted input.

`35.0` adds an `include` directory containing the well-known-type `.proto`
files to the `Google.Protobuf.Tools` NuGet package. Packaged compiler
invocations can resolve those imports from the package instead of requiring a
separate copy.

`36.0-rc1` adds C# support for Edition 2026 and moves nullable-reference-type
generation into that edition. Select Edition 2026 when generated C# nullability
is required, then rebuild with nullable diagnostics enabled.

The compiler warns on deprecated `cc_generic_service` and
`py_generic_service` alongside `java_generic_service`; remove any shared
schema options rather than fixing only the Java one.

## Objective-C unknown fields

The first Objective-C breaking runtime in `30.0-migration` moves from 3.x to
4.30.x and replaces `GPBUnknownFieldSet` with ordering-preserving
`GPBUnknownFields`.

A `GPBUnknownField` now represents one value instead of collecting values by
field number. Use:

- `initFromMessage:` to extract unknown fields;
- `mergeUnknownFields:extensionRegistry:error:` to update a message; and
- `clearUnknownFields` to remove them.

Review any code that assumed values were grouped or that enumeration could
discard original ordering.

## Objective-C removed runtime APIs

The `30.0-migration` replacements are:

| Removed or obsolete API | Replacement |
| --- | --- |
| `mergeFrom:extensionRegistry:` | `mergeFrom:extensionRegistry:error:` |
| `GPBDuration.timeIntervalSince1970` | `GPBDuration.timeInterval` |
| `GPBTextFormatForUnknownFieldSet()` | `GPBTextFormatForMessage()` |
| `GPBFileDescriptor.syntax` | No longer query this obsolete property |

Runtime entry points for generated code older than 3.22 are removed. Regenerate
those files with a current compiler before updating the Objective-C runtime.

## Objective-C nullability and descriptors

`34.0-announcement` corrects `GPB*Dictionary` nullability. Affected Swift
callers now receive `Optional<T>` and must unwrap or propagate absence.

The same change removes `-[GPBFieldDescriptor optional]`. Test:

```text
!required && fieldType == GPBFieldTypeSingle
```

This is a cardinality test; use an appropriate presence semantic when the
question is whether a singular field tracks presence.

## Objective-C generated extensions and oneofs

`34.0` adds three code-generation modes for proto extensions. Select and test
the intended mode explicitly where build size, registration, or linking
depends on generated extension output.

The same release emits oneof presence-checking accessors. Prefer those
generated accessors over hand-derived checks against the oneof case.

## Upgrade checks

- Regenerate Java after file-layout, large-enum, or collision-related changes.
- Compare strict and default Java JSON parsing and sorted-key output.
- Exercise deep JSON and invalid UTF-8 failure paths in Java or C# as
  applicable.
- Resolve C# well-known imports from the package's `include` directory in a
  clean build.
- Regenerate Objective-C code older than 3.22 before changing the runtime.
- Preserve unknown-field ordering and test merge failures.
- Compile Swift consumers with the corrected Objective-C optionality.
- Verify extension registration and oneof presence with regenerated output.
