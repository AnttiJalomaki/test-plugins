# Java, C#, and Objective-C

## Java generated APIs

Edition 2024's `nest_in_file_class` feature replaces `java_multiple_files`.
Classes are emitted in separate files by default, and the outer class defaults
to the camel-cased proto filename plus `Proto`; override it with
`java_outer_classname`. (edition-2024-announcement)

The opt-in `large_enum` feature permits more constants than a normal Java enum
but produces an enum-like type that does not support all enum operations,
including `switch`. At 34.0, the lite runtime also honors `large_enum` and
handles aliased large-enum values correctly. (edition-2024-announcement, 34.0)

Generated `isInitialized()` accessors are deprecated at 34.0 for message types
without required fields. Calls on those types can produce new deprecation
diagnostics. (34.0)

## Java and C# recursion limits

At 34.0, recursion enforcement expands to Java JSON `Any`-inside-`Any` nesting
and C# JSON well-known types containing deeply nested arrays. Previously
accepted deep inputs can be rejected; do not treat recursion-limit failures as
generic parse regressions. (34.0)

## C# runtime and tooling

C# surfaces UTF-8 errors earlier when a protobuf `string` contains invalid
UTF-8. Validate byte sources before assigning or parsing string fields.
(30.0-migration)

At 35.0, the `Google.Protobuf.Tools` NuGet package includes an `include`
directory containing the well-known-type `.proto` files. Packaged compiler
invocations can resolve these imports directly from the package.
(35.0)

## Objective-C runtime major and unknown fields

Objective-C's first breaking runtime moves from 3.x to 4.30.x in the 30.0
migration. It replaces `GPBUnknownFieldSet` with ordering-preserving
`GPBUnknownFields`; each `GPBUnknownField` represents one value rather than all
values for a field number. (30.0-migration)

Use `initFromMessage:` to extract unknown fields,
`mergeUnknownFields:extensionRegistry:error:` to apply updates, and
`clearUnknownFields` to remove them. (30.0-migration)

## Removed Objective-C APIs and old gencode

Apply these replacements from the 30.0 migration:

| Removed or obsolete | Replacement |
| --- | --- |
| `mergeFrom:extensionRegistry:` | `mergeFrom:extensionRegistry:error:` |
| `GPBDuration.timeIntervalSince1970` | `GPBDuration.timeInterval` |
| `GPBTextFormatForUnknownFieldSet()` | `GPBTextFormatForMessage()` |
| `GPBFileDescriptor.syntax` | No syntax query through this obsolete API |

Runtime entry points for gencode older than 3.22 were removed; regenerate old
Objective-C generated files with a current compiler. (30.0-migration)

At the v34 boundary, corrected `GPB*Dictionary` nullability makes affected
Swift return values `Optional<T>`. `-[GPBFieldDescriptor optional]` is removed;
test `!required && fieldType == GPBFieldTypeSingle` instead.
(34.0-announcement)

## Objective-C generation additions

At 34.0, Objective-C code generation supports three modes for proto extension
generation and emits oneof presence-checking accessors. Select the extension
mode deliberately and prefer generated oneof-presence APIs over inspecting
storage details. (34.0)
