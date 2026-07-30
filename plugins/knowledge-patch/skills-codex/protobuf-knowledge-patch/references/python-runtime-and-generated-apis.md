# Python Runtime and Generated APIs

Use this reference when upgrading the Python runtime, replacing dynamic-message
or reflection APIs, validating assignments, integrating upb containers, or
hardening text and descriptor parsing.

## Runtime baselines and generated-code compatibility

`30.0-migration` moves the Python package from major 5.29.x to 6.30.x and
requires Python 3.9 or newer. `34.0-migration` raises the interpreter minimum
to Python 3.10.

The provisional boundary in `34.0-announcement` moves C++ and Python from 6.33
to 7.34.0. Python generated output does not change for 7.34.x, and poison
checks are relaxed so older generated files do not warn or error merely
because of this runtime package-major change.

Under `release-lifecycle`, generated code from Python 3.20.0 onward is
descriptor-based and supported through at least runtime 8.x. A future major
that breaks the extended window is expected to introduce poison warnings and
errors in advance. Still regenerate on each update rather than treating the
window as a reason to leave generated files stale.

## Bazel Python rules

The `30.0-migration` build changes are:

- `bazel/system_python.bzl` is removed. Prefer `protobuf_deps.bzl`, or refer to
  its moved path at `python/dist/system_python.bzl`.
- The internal `py_proto_library` from `protobuf.bzl` is removed. Use the
  official rule under `bazel/py_proto_library`.

`36.0-rc1` deprecates `internal_py_proto_library` ahead of Q1 2027 breaking
changes. Move affected targets to the supported Python proto rule.

## Dynamic messages and removed runtime APIs

`30.0-migration` removes:

- `reflection.ParseMessage`
- `reflection.MakeClass`
- prototype and creation methods on `MessageFactory` and `SymbolDatabase`
- `MessageFactory.GetMessages`
- `SymbolDatabase.GetMessages`

Use `message_factory.GetMessageClass()` for a descriptor, or
`message_factory.GetMessageClassesForFiles()` when starting from files.

The legacy `service` RPC interfaces are also removed; generate RPC bindings
with the appropriate RPC-specific plugin. The C++-extension-only
`GetDebugString` has no replacement.

## Assignment and conversion validation

Closed-enum setters in Python and upb reject invalid enum values under Edition
2023 starting with `30.0-migration`.

`34.0-announcement` makes scalar assignment stricter:

- A `bool` assigned to an integer or enum field is rejected rather than
  implicitly converted.
- Converting an invalid type to `Timestamp` or `Duration` raises `TypeError`
  rather than `AttributeError`.

Update exception handlers to catch the new type and add negative tests for
boolean inputs that previously behaved like integers.

Message construction through keyword arguments becomes more reliable in
`34.0`: errors involving repeated-field initialization are no longer silently
swallowed. Invalid repeated values can now fail initialization.

## Map and repeated-field containers

Since `30.0-migration`, `ScalarMap.setdefault` requires both a key and a value.
Calling `setdefault` is rejected entirely for message-valued maps; use explicit
lookup and assignment semantics.

`34.0` adds a NumPy binding for scalar repeated fields, enabling direct
array-oriented interoperability.

`36.0-rc1` lets repeated scalar fields consume objects supported by the Python
Buffer API. Validate element types, item sizes, and ownership for the selected
buffer exporter rather than assuming every buffer has a compatible layout.

## Generated class identity

Nested generated classes in `30.0-migration` include the containing message in
`__qualname__` but keep the short `__name__`. For example:

```python
Foo.Bar.__name__ == "Bar"
Foo.Bar.__qualname__ == "Foo.Bar"
```

Code that serializes, registers, logs, or dynamically imports classes by
qualified name must account for the outer message.

## Formatting options removed

`34.0-announcement` removes:

- JSON serialization's deprecated `float_precision` option.
- Text-format serialization's `float_format` option.
- Text-format serialization's `double_format` option.

Remove the keyword arguments and, where exact presentation matters, compare
the new formatter's result in golden tests.

## Recursion and malformed input

`34.0` broadens recursion guards for nested messages in Python and upb. Deep
inputs that bypassed previous limits can now fail.

`35.0` adds an optional recursion-depth limit to Python `text_format`. Set the
limit when parsing untrusted or deeply nested textual protos.

Treat recursion failure as a normal parse error and test the boundary just
below, at, and above the configured limit.

## Free-threaded Python and upb

`35.0` adds free-threaded Python support to the upb runtime and fixes races in
lazy message initialization and repeated-field presence handling. When using
a free-threaded interpreter:

- run concurrent initialization and mutation tests;
- retest code that inferred synchronization from the global interpreter lock;
  and
- update the runtime as a unit rather than mixing old native extensions.

## Descriptor and option APIs

upb performs stricter `syntax` and `edition` checks while parsing descriptors
in `34.0`. Malformed dynamically supplied descriptors that previously loaded
can now be rejected.

For scalar types in `36.0-rc1`, `GetOptions()` returns immutable options under
Python/upb. Mutating the returned object raises `TypeError`; construct or copy
the desired descriptor data instead of modifying the shared options instance.

`DescriptorDatabase.FindFileContainingSymbol()` accepts a leading `.` on a
fully qualified symbol name in `36.0-rc1`, matching `DescriptorPool` and other
language runtimes.

The same release adds the C API:

```cpp
PyDescriptorPool_FromSharedPool(std::shared_ptr)
```

C++ extensions can wrap a descriptor pool in Python while preserving shared
ownership of the underlying pool.

## Upgrade checks

- Verify the interpreter baseline before resolving the runtime wheel.
- Replace reflection and dynamic-message creation APIs before upgrading.
- Exercise invalid enum, integer, timestamp, duration, map, and repeated-field
  assignments.
- Remove obsolete float-formatting options.
- Configure and test recursion limits for untrusted text.
- Test native extensions and container access under free-threaded execution.
- Stop mutating scalar option objects returned by upb.
- Test both dotted and undotted fully qualified descriptor lookups when the
  application accepts either form.
