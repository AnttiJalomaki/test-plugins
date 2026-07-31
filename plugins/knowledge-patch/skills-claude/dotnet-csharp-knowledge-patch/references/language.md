# Language Features and Compiler Compatibility

## C# language features (`10.0-guides`)

### Extension members

C# 14 extension blocks can add instance properties and methods, static
properties and methods, and operators to an extended type. Name the receiver
to define instance members; omit the receiver name to permit static members.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

### Field-backed properties

Within a property accessor, the contextual `field` token names the
compiler-synthesized backing field. It enables validation without a separate
field declaration.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

If the containing type already declares an identifier named `field`, use
`@field` to identify it by name or `this.field` to access the instance member.

### First-class span conversions

C# 14 adds implicit conversions among arrays, `Span<T>`, and
`ReadOnlySpan<T>`. Span types also participate more naturally as extension
receivers, in composed conversions, and in generic type inference. These rules
can change overload selection, so test overload-sensitive calls after switching
the language version.

### Unbound generic types in `nameof`

`nameof` accepts an unbound generic type. `nameof(List<>)` evaluates to
`"List"`; no type argument is required.

### Modifiers on implicitly typed lambda parameters

Simple lambda parameters may use `scoped`, `ref`, `in`, `out`, or
`ref readonly` without explicit parameter types. `params` still requires an
explicitly typed parameter list.

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

### Partial constructors and events

An instance constructor or event may be partial. It must have exactly one
defining declaration and one implementing declaration. Only the implementing
constructor can specify `this()` or `base()`. The implementing declaration of a
partial field-like event supplies its `add` and `remove` accessors.

### User-defined compound assignment

A type can declare user-defined compound-assignment operators. Use these when
compound assignment needs dedicated behavior instead of the behavior derived
from the corresponding binary operator.

### Null-conditional assignment

`?.` and `?[]` may appear on the left side of simple and compound assignments.
The right side is evaluated only when the receiver is non-null. Null-conditional
`++` and `--` are not supported.

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

## Visual Basic compiler compatibility (`10.0`)

The Visual Basic compiler interprets and enforces `unmanaged` generic
constraints and honors `OverloadResolutionPriorityAttribute`. This enables
Span-oriented overload selection and resolves ambiguities consistently with
the newer runtime APIs.
