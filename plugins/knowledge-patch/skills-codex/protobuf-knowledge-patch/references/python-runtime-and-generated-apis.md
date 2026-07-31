# Python Runtime and Generated APIs

## Runtime and interpreter baselines

The Python package major moved from 5.29.x to 6.30.x in the 30.0 migration and
requires Python 3.9 or newer. The 34.0 migration raises the interpreter minimum
to Python 3.10. (30.0-migration, 34.0-migration)

The shared v34 release moves the Python package to 7.34.0 after 6.33, but its
gencode does not change for 7.34.x. Poison checks are relaxed so old generated
files do not produce compatibility warnings or errors. (34.0-announcement)

Python gencode from 3.20.0 onward is descriptor-based and supported through at
least runtime 8.x. Even so, regenerate on release updates and never run gencode
against an older runtime than the compiler/plugin that produced it.
(release-lifecycle)

## Bazel rules

The 30.0 migration removes the `bazel/system_python.bzl` alias. Prefer
`protobuf_deps.bzl`, or use the moved
`python/dist/system_python.bzl` location. The internal `py_proto_library` from
`protobuf.bzl` is also removed; use the official rule under
`bazel/py_proto_library`. (30.0-migration)

## Removed reflection, factory, and service APIs

The 30.0 migration removes:

- `reflection.ParseMessage` and `reflection.MakeClass`;
- prototype and creation methods on `MessageFactory` and `SymbolDatabase`;
- their `GetMessages` methods;
- legacy generic `service` RPC interfaces;
- the C++-extension-only `GetDebugString`.

Use `message_factory.GetMessageClass()` or
`GetMessageClassesForFiles()` for dynamic message classes. Replace generic
service interfaces with RPC-specific generator plugins. `GetDebugString` has
no replacement. (30.0-migration)

Python `FieldDescriptor.label` is removed in the 34.0 migration. Use
`isRepeated`, `isRequired`, and `hasPresence` semantics rather than label
constants. (34.0-migration)

## Maps, enums, and assignment validation

`ScalarMap.setdefault` requires both key and value. Message-valued maps reject
`setdefault` entirely because a default message cannot be inserted through
that scalar-oriented operation. (30.0-migration)

Python and upb setters reject invalid values for closed enums under Edition
2023. At the v34 boundary they also reject `bool` assignment to enum or integer
fields instead of converting it implicitly. (30.0-migration,
34.0-announcement)

Invalid-type conversion to `Timestamp` or `Duration` raises `TypeError` rather
than `AttributeError`; update exception handlers accordingly.
(34.0-announcement)

Message construction by keyword no longer swallows certain repeated-field
errors. Invalid repeated-field initializers can now raise an exception.
(34.0)

## Generated class identity

Generated nested message classes include the outer message in `__qualname__`
while retaining the short `__name__`. For example, `Foo.Bar.__qualname__` is
`"Foo.Bar"`, while `Foo.Bar.__name__` remains `"Bar"`. Code using qualified
names for registration, pickling, or diagnostics must account for this.
(30.0-migration)

## Serialization and parsing

The JSON serializer no longer accepts deprecated `float_precision`; text
format serialization no longer accepts `float_format` or `double_format`.
Remove those keyword arguments. (34.0-announcement)

At 34.0, Python and upb add guards for nested messages as part of broader
recursion-limit enforcement. At 35.0, `text_format` adds an optional recursion
depth limit; set it when parsing untrusted or deeply nested text input.
(34.0, 35.0)

## Repeated scalars and NumPy/buffer interoperability

At 34.0, repeated scalar fields gain a NumPy binding for direct array-oriented
interoperability. Use it instead of element-by-element conversion where an
array workflow is appropriate. (34.0)

## upb and free-threaded Python

At 34.0, upb validates descriptor `syntax` and `edition` more strictly.
Malformed dynamic descriptors accepted by older versions can fail parsing.
upb generators also enable Edition 2024. (34.0)

At 35.0, the Python upb runtime supports free-threaded Python and fixes races
in lazy message initialization and repeated-field presence handling. Prefer
this or a later release before evaluating protobuf code under free-threaded
execution. (35.0)
