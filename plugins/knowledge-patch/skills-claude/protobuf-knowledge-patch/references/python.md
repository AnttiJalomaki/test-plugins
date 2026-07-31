# Python and upb runtime

## Runtime baseline (`30.0-migration`)

The Python package major moves from 5.29.x to 6.30.x and requires Python 3.9 or
newer. Protobuf `34.0-migration` raises the interpreter minimum again to Python
3.10.

## Closed-enum assignment (`30.0-migration`)

Python and upb setters reject values that are invalid for closed enums under
Edition 2023. Validate integer-to-enum conversions before assignment rather
than depending on an unknown numeric value being stored.

## Removed dynamic-message APIs (`30.0-migration`)

The following are removed:

- `reflection.ParseMessage` and `reflection.MakeClass`;
- prototype and creation methods on `MessageFactory` and `SymbolDatabase`; and
- the corresponding `GetMessages` methods.

Use `message_factory.GetMessageClass()` for one descriptor or
`GetMessageClassesForFiles()` for a file set. Replace legacy `service` RPC
interfaces with an RPC-specific generator plugin. The C++-extension-only
`GetDebugString` has no replacement.

## Map `setdefault` (`30.0-migration`)

`ScalarMap.setdefault` requires both key and value. Message-valued maps reject
`setdefault` entirely; initialize the entry through the message-map API instead.

## Nested class qualification (`30.0-migration`)

Generated nested message classes include the outer message in `__qualname__`
while preserving the short `__name__`. For example, `Foo.Bar.__qualname__` is
`"Foo.Bar"`, while `Foo.Bar.__name__` remains `"Bar"`. Update reflection,
pickling assumptions, and name-based tests accordingly.

## Scalar assignment and WKT conversions (`34.0-announcement`)

Assigning `bool` to an enum or integer field is rejected instead of converting
it implicitly. Invalid-type conversion to `Timestamp` or `Duration` raises
`TypeError` rather than `AttributeError`; exception handlers must catch the
newly correct type.

## Removed formatting options (`34.0-announcement`)

The JSON serializer no longer accepts deprecated `float_precision`. Text-format
serialization no longer accepts `float_format` or `double_format`. Remove these
arguments and validate output using the runtime's standard float rendering.

## Descriptor label removal (`34.0-migration`)

`FieldDescriptor.label` is removed. Use `is_repeated`, `is_required`, and
presence-oriented APIs rather than reconstructing a legacy label.

## Repeated-field initialization and NumPy (`34.0`)

Scalar repeated fields expose a NumPy binding for direct array-oriented
interoperability.

Message construction through keyword arguments no longer suppresses some
errors involving repeated fields. Invalid repeated-field initialization can now
raise immediately; do not treat construction as successful until it returns.

## Descriptor parsing and Editions in upb (`34.0`)

upb performs additional validation of descriptor `syntax` and `edition`, so
malformed dynamic descriptor data can be rejected. upb generators also enable
Edition 2024.

## Recursion limits (`34.0`, `35.0`)

Python and upb add recursion guards for nested messages. Python `text_format`
also accepts an optional recursion-depth limit. Set it when parsing untrusted or
deeply nested text to bound recursive work, and handle limit exhaustion as a
normal parse failure.

## Free-threaded Python (`35.0`)

The upb runtime supports free-threaded Python. The release fixes races in lazy
message initialization and repeated-field presence handling that affected that
mode. Exercise concurrent initialization and mutation paths when enabling a
free-threaded interpreter.
