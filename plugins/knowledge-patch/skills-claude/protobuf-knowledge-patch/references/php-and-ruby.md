# PHP, Ruby, and JRuby runtimes

Source batches represented here: 30.0-migration, 30.0, 31.0,
34.0-announcement, 34.0-migration, 34.0, 36.0-rc1.

## PHP runtime and type baselines

Protobuf v34 requires PHP 8.2 or newer. The shared v34 release moves the PHP
runtime package from 4.33 to 5.34.

Replace removed PHP runtime types:

| Removed | Replacement |
| --- | --- |
| `Google\Protobuf\Field_Kind` | `Google\Protobuf\Field\Kind` |
| `Google\Protobuf\Field_Cardinality` | `Google\Protobuf\Field\Cardinality` |
| `Google\Protobuf\Internal\RepeatedField` | `Google\Protobuf\RepeatedField` |

Generated setters carry PHP type hints in v34, and redundant `GPBUtil` checks
are removed. Reflection, subclasses, and callers that assumed untyped
signatures need adjustment.

## PHP reflection and defaults

`FieldDescriptor::getLabel()` is deprecated and then removed in v34. PHP field
descriptors implement `hasPresence()` in v34, while the broken
`hasOptionalKeyword()` helper is removed. Use cardinality and presence
semantics directly.

The PHP runtime honors proto2 and Editions scalar defaults at the v34 boundary
instead of silently ignoring them. Pure-PHP type validation aligns with
upb-PHP, including rejection of `null` for string fields.

## PHP JSON and binary validation

During JSON parsing, the v30 migration changes nonnumeric strings such as
`""`, `"12abc"`, and `"abc"` for numeric fields from warnings to errors.

At the v34 boundary, PHP JSON parsing also rejects:

- out-of-range numbers;
- non-integer numeric values for integer fields;
- duplicate fields belonging to one oneof;
- non-string values for string fields.

JSON serialization rejects `Infinity` and `NaN` as number values. A v34
serializer option can emit fields whose values equal their defaults.

In 36.0-rc1, `mergeFromString` and `serializeToString` accept a configurable
`recursion_limit`, allowing applications to bound deeply nested binary
traversal.

## Ruby runtime support and behavior

Ruby 3.0 support is removed in v31, making Ruby 3.1 the minimum. Ruby 4.0 is
supported in v34. The 36.0-rc1 line removes Ruby 3.1 support, so upgrade the
interpreter before adopting it.

Ruby surfaces UTF-8 enforcement errors earlier for invalid bytes in protobuf
`string` fields after the v30 migration.

Ruby JSON numeric parsing rejects nonnumeric strings such as `""`, `"12abc"`,
and `"abc"` after v30; the same inputs only warned in v29.x.

Ruby code generation can emit RBS files in v34 so generated protobuf APIs can
participate in RBS-based type checking.

## JRuby

JRuby uses its FFI implementation by default beginning in v30. Applications
that depended on the old implementation can break. This does not trigger a
Ruby package major bump because JRuby support is best-effort rather than
official.

The support target is the newest JRuby compatible with protobuf's minimum
supported Ruby version.
