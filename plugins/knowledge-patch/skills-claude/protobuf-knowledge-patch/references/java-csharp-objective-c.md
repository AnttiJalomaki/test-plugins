# Java, C#, and Objective-C runtimes

## Objective-C unknown fields (`30.0-migration`)

Objective-C runtime 4.30 replaces `GPBUnknownFieldSet` with the
ordering-preserving `GPBUnknownFields`. Each `GPBUnknownField` represents one
value instead of grouping all values with the same field number.

- Extract unknown fields with `initFromMessage:`.
- Update them with `mergeUnknownFields:extensionRegistry:error:`.
- Remove them with `clearUnknownFields`.

## Objective-C API removals (`30.0-migration`)

- Replace `mergeFrom:extensionRegistry:` with
  `mergeFrom:extensionRegistry:error:` and handle its failure.
- Replace `GPBDuration.timeIntervalSince1970` with `GPBDuration.timeInterval`.
- Replace `GPBTextFormatForUnknownFieldSet()` with
  `GPBTextFormatForMessage()`.
- Stop using the obsolete `GPBFileDescriptor.syntax` property.

Runtime entry points for generated code older than 3.22 are removed. Regenerate
such Objective-C outputs with a current compiler before adopting the newer
runtime.

## Earlier UTF-8 failures (`30.0-migration`)

C# now surfaces UTF-8 enforcement errors earlier when a protobuf `string`
contains invalid UTF-8. Do not depend on invalid data surviving until a later
serialization step.

## Objective-C Swift nullability (`34.0-announcement`)

Corrected nullability on `GPB*Dictionary` APIs changes affected Swift results to
`Optional<T>`. Update call sites to unwrap or propagate the optional value.

`-[GPBFieldDescriptor optional]` is removed. Express the equivalent test as:

```text
!required && fieldType == GPBFieldTypeSingle
```

## Java initialization checks (`34.0`)

Generated `isInitialized()` accessors are deprecated for message types that do
not contain required fields. Calls on those types can newly trigger deprecation
diagnostics; remove them or restrict checks to messages whose required-field
semantics need them.

## Java large enums (`34.0`)

The Java lite runtime now honors the `large_enum` feature and correctly handles
large enums containing aliased values. These generated enum-like types still do
not support every ordinary Java enum operation, including `switch`.

## Recursion-limit coverage (`34.0`)

Java JSON parsing now applies recursion limits to `Any` containing nested
`Any` values. C# JSON parsing also guards well-known types that contain deeply
nested arrays. Inputs that previously bypassed recursion accounting can now be
rejected; keep limits and failure handling in untrusted-input tests.

## Objective-C generated extensions and oneofs (`34.0`)

Objective-C code generation supports three modes for proto extension
generation. It also emits presence-checking accessors for oneofs. Generator
wrappers should choose the intended extension mode explicitly and use the new
accessors rather than inferring oneof presence indirectly.

## C# packaged well-known types (`35.0`)

The `Google.Protobuf.Tools` NuGet package includes an `include` directory with
the well-known-type `.proto` files. Compiler invocations sourced from the
package can resolve those imports directly from that directory.
