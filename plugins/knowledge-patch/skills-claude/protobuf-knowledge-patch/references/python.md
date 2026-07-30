# Python and upb runtime

Source batches represented here: 30.0-migration, 31.0, 34.0-announcement,
34.0-migration, 34.0, 35.0, 36.0-rc1.

## Runtime baselines and compatibility

The Python package major moves from 5.29.x to 6.30.x in the v30 migration and
requires Python 3.9 or newer. Protobuf v34 raises the interpreter minimum to
Python 3.10 and moves the runtime package to 7.34.x.

Python gencode from 3.20.0 onward is descriptor-based and is supported through
at least runtime 8.x. Never pair generated code with a runtime older than the
compiler/plugin that produced it.

## Dynamic messages and removed APIs

The v30 migration removes:

- `reflection.ParseMessage`
- `reflection.MakeClass`
- prototype/creation methods on `MessageFactory` and `SymbolDatabase`
- `GetMessages` methods on those factory/database classes

Use module-level `message_factory.GetMessageClass()` or
`message_factory.GetMessageClassesForFiles()`.

Legacy `service` RPC interfaces are removed; use an RPC-specific generator
plugin. The C++-extension-only `GetDebugString` has no replacement.

`FieldDescriptor.label` is deprecated in v31 and removed in v34. Use
`is_repeated`, `is_required`, and `has_presence`-style semantics rather than
one label value.

## Assignment and construction

Closed-enum field setters reject invalid values under Edition 2023 beginning
with the v30 migration.

`ScalarMap.setdefault` requires both key and value. Message-valued maps reject
`setdefault` entirely.

Generated nested classes keep a short `__name__`, but `__qualname__` includes
the outer message. For example, `Foo.Bar.__name__` is `"Bar"` and
`Foo.Bar.__qualname__` is `"Foo.Bar"`.

At the v34 boundary, assigning `bool` to an enum or integer field is rejected
instead of coercing it. Invalid-type conversion to `Timestamp` or `Duration`
raises `TypeError`, not `AttributeError`; update exception handlers.

Message construction from keyword arguments no longer swallows some
repeated-field errors in v34. Invalid repeated-field initialization can now
raise.

## Formatting and recursion

The v34 JSON serializer removes the deprecated `float_precision` option.
Text-format serialization removes `float_format` and `double_format`.

Python `text_format` gains an optional recursion-depth limit in v35. Set it
when parsing untrusted or deeply nested TextFormat.

Python and upb also add guards for nested-message recursion in v34. Inputs that
previously bypassed recursion checks can be rejected.

## Repeated scalar interoperability

Python repeated scalar fields gain a NumPy binding in v34.

In 36.0-rc1, repeated scalar fields can be assigned from objects supported by
the Python Buffer API, enabling compatible buffer-exporting containers without
an intermediate list.

## Descriptor and options behavior

upb performs stricter validation of descriptor `syntax` and `edition` in v34.
Malformed dynamic descriptor data can fail where it was previously accepted.

For scalar types, `GetOptions()` returns immutable options under Python/upb in
36.0-rc1. Mutating the returned object raises `TypeError`.

`DescriptorDatabase.FindFileContainingSymbol()` accepts a fully qualified
symbol beginning with `.` in 36.0-rc1, aligning it with `DescriptorPool` and
other runtimes.

The Python C API adds
`PyDescriptorPool_FromSharedPool(std::shared_ptr)` in 36.0-rc1. A C++
extension can expose a Python descriptor pool while retaining shared ownership
of the underlying pool.

## Free-threaded Python

The upb runtime supports free-threaded Python in v35. The same release fixes
races in lazy message initialization and repeated-field presence handling that
affected free-threaded use.

Audit extension code and tests for actual free-threaded safety rather than
assuming runtime support makes surrounding application code safe.
