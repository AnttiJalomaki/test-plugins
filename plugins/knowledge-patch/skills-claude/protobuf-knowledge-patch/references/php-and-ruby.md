# PHP, Ruby, and JRuby runtimes

## Strict numeric JSON parsing (`30.0-migration`)

Ruby and PHP now reject nonnumeric strings such as `""`, `"12abc"`, and
`"abc"` for numeric protobuf fields during JSON parsing. Version 29.x only
warned for these cases. Treat the parse as a validation failure and do not rely
on partial numeric conversion.

## Earlier Ruby UTF-8 failures (`30.0-migration`)

Ruby surfaces UTF-8 enforcement errors earlier when a protobuf `string` field
contains invalid UTF-8. Validate or transcode input before assignment rather
than expecting serialization to be the first failure point.

## JRuby implementation default (`30.0`)

JRuby uses its FFI implementation by default. Applications that depended on the
previous implementation must test native integration and deployment packaging.
The behavior did not trigger a Ruby package-major bump because JRuby remains a
best-effort target.

## Ruby interpreter baseline (`31.0`)

The Ruby runtime requires Ruby 3.1 or newer.

## PHP runtime type replacements (`34.0-announcement`)

- Replace `Google\Protobuf\Field_Kind` with
  `Google\Protobuf\Field\Kind`.
- Replace `Google\Protobuf\Field_Cardinality` with
  `Google\Protobuf\Field\Cardinality`.
- Replace `Google\Protobuf\Internal\RepeatedField` with
  `Google\Protobuf\RepeatedField`.

## PHP JSON validation (`34.0-announcement`)

PHP JSON parsing rejects:

- numeric values outside the target field's range;
- non-integer numeric values for integer fields;
- duplicate fields from the same oneof; and
- non-string JSON values for string fields.

Serialization also rejects `Infinity` and `NaN` when presented as JSON number
values. Route all these cases through normal parse/serialize error handling.

## PHP defaults and type consistency (`34.0-announcement`)

The runtime now honors proto2 and Editions scalar defaults instead of ignoring
them. Pure-PHP type checks align with upb-PHP, including rejection of `null` for
string fields. Audit code that used an absent scalar's zero value instead of its
declared default.

## PHP runtime floor and descriptor labels (`34.0-migration`)

PHP requires version 8.2 or newer. The deprecated
`FieldDescriptor::getLabel()` accessor is removed; use cardinality and presence
predicates.

## Typed generated setters (`34.0`)

Generated PHP setters carry PHP type hints, and redundant `GPBUtil` checks are
removed. Reflection, subclasses, mocks, or callers that depended on untyped
setter signatures must align with the generated types.

## PHP JSON default emission (`34.0`)

JSON serialization can opt into emitting fields whose values equal their
defaults. Decide explicitly whether external JSON contracts expect these fields
and update golden output when enabling the option.

## PHP presence reflection (`34.0`)

PHP field descriptors implement `hasPresence()`. The broken
`hasOptionalKeyword()` helper is removed. Reflection logic should ask about
presence semantics directly rather than infer them from an optional keyword.

## Ruby RBS output and Ruby 4 (`34.0`)

Ruby code generation can emit RBS files for generated protobuf types. Add the
outputs to the type-checking pipeline when using RBS. The runtime also supports
Ruby 4.0.
