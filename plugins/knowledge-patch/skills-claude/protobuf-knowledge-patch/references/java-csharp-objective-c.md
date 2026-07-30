# Java, C#, and Objective-C runtimes

Source batches represented here: 30.0-migration, edition-2024-announcement,
34.0-announcement, 34.0, 35.0, 36.0-rc1.

## Java generation and reflection

Edition 2024 removes `java_multiple_files` in favor of the
`nest_in_file_class` feature. Classes are generated into their own files by
default. The default outer class is the camel-cased proto filename plus
`Proto`; use `java_outer_classname` for an explicit name.

The Edition 2024 `large_enum` feature opts into enum-like generated types
beyond Java's ordinary enum constant limit. These do not support every enum
operation, including `switch`. Java lite honors this feature in v34 and
correctly handles aliased large-enum values.

Generated `isInitialized()` is deprecated in v34 for message types without
required fields. Remove redundant calls or tolerate deprecation diagnostics
only while supporting older output.

The experimental Java `FieldOrder` enum is removed in 36.0-rc1. Stop compiling
against it.

Java warns about potential `OneofDescriptor` collisions in 36.0-rc1 ahead of
Q1 2027 breaking changes. Treat the warning as a schema naming/regeneration
task.

## Java JSON and timestamp parsing

Java `JsonFormat` gains an opt-in strict JSON parser mode in 36.0-rc1. Select
it when strict validation is part of the input contract.

When sorted map output is requested, `JsonFormat` now compares keys using
natural Java `String` ordering instead of UTF-8 `ByteString` ordering. Golden
JSON output can change even though map contents do not.

`Timestamps.parse()` attempts non-lenient parsing first and warns if it must
fall back to lenient parsing. Normalize inputs that only the lenient parser
accepts.

Java JSON recursion limits now cover nested `Any` values containing `Any`.
Deep payloads that escaped earlier checks can fail.

## C# behavior

C# surfaces UTF-8 enforcement errors earlier for invalid bytes in protobuf
`string` fields beginning with the v30 migration.

Recursion-limit enforcement expands in v34 to C# JSON well-known types
containing deep arrays.

The `Google.Protobuf.Tools` package includes well-known-type `.proto` files
under `include` in v35.

C# advertises Edition 2026 support in 36.0-rc1, and nullable-reference-type
generation belongs to that edition. Select Edition 2026 when generated
nullability is required.

## Objective-C unknown fields

Objective-C's first breaking runtime line moves from 3.x to 4.30.x in the v30
migration. `GPBUnknownFields` replaces `GPBUnknownFieldSet` and preserves
unknown-field ordering. A `GPBUnknownField` represents one value instead of
grouping all values by field number.

Use:

- `initFromMessage:` to extract unknown fields;
- `mergeUnknownFields:extensionRegistry:error:` to update them;
- `clearUnknownFields` to remove them.

## Objective-C removed and changed APIs

At the v30 boundary:

- replace `mergeFrom:extensionRegistry:` with
  `mergeFrom:extensionRegistry:error:`;
- replace `GPBDuration.timeIntervalSince1970` with
  `GPBDuration.timeInterval`;
- replace `GPBTextFormatForUnknownFieldSet()` with
  `GPBTextFormatForMessage()`;
- stop using obsolete `GPBFileDescriptor.syntax`.

Runtime entry points for generated code older than 3.22 are removed.
Regenerate those files with a current compiler.

At the v34 boundary, corrected `GPB*Dictionary` nullability makes affected
Swift return values `Optional<T>`. `-[GPBFieldDescriptor optional]` is removed;
test `!required && fieldType == GPBFieldTypeSingle`.

Objective-C codegen gains three modes for proto extension generation in v34
and emits oneof presence-checking accessors.
