# Language and Compiler Features

This reference groups language work rather than release chronology. The
relevant extraction batch identifiers are `10.0-guides` and `10.0`.

## Extension Members

C# 14 extension blocks extend a type with more than instance methods. They can
declare instance properties and methods, static properties and methods, and
operators.

A named receiver makes the members in the block instance members:

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

Omit the receiver name when declaring static extension members. Keep the block
inside a static containing class, and choose the form according to whether a
member needs an instance receiver.

## Field-Backed Properties

The contextual `field` token accesses a compiler-synthesized property backing
field. It lets an accessor add behavior without declaring and maintaining a
separate field:

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

`field` is contextual. In a type that already declares an identifier named
`field`, use `@field` or `this.field` when referring to that existing member.
Review such types carefully during migration because an unqualified token in
an accessor can acquire the new contextual meaning.

## First-Class Span Conversions

C# 14 adds implicit conversions among arrays, `Span<T>`, and
`ReadOnlySpan<T>`. Span types also participate more naturally in composed
conversions, extension receiver binding, and generic type inference.

This changes more than convenience syntax. Overload resolution can select a
different method when new span conversions make candidates applicable or
change which conversion is better. Recompile and test overload-heavy APIs,
especially APIs that expose both array and span forms.

## Unbound Generics in `nameof`

`nameof` accepts an unbound generic type. Supply the arity commas but no type
arguments:

```csharp
string name = nameof(List<>); // "List"
```

This avoids inventing a closed constructed type merely to retrieve the generic
type's simple name.

## Modifiers on Implicitly Typed Lambda Parameters

Simple lambda parameters can use `scoped`, `ref`, `in`, `out`, or
`ref readonly` without explicit parameter types:

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

`params` is the exception: it still requires an explicitly typed parameter
list. Preserve modifier agreement with the target delegate and use explicit
types when inference does not communicate the intended signature clearly.

## Partial Constructors

An instance constructor can be partial. It has exactly one defining
declaration and one implementing declaration. Only the implementing
constructor may provide a `this()` or `base()` initializer. This makes the
initializer part of the implementation rather than generated declaration
metadata.

## Partial Events

An event can be partial, again with exactly one defining and one implementing
declaration. The defining declaration is field-like. The implementing event
provides the `add` and `remove` accessors. Do not supply competing
implementations or accessors on the defining half.

## User-Defined Compound Assignment

A type can declare dedicated compound-assignment operators. This allows
`value += other` and related operations to have purpose-built behavior instead
of always being synthesized from the corresponding binary operator and an
assignment conversion. When a mutable or high-performance type defines both,
do not assume the compound form is semantically identical to `value = value +
other`.

## Null-Conditional Assignment

Null-conditional element and member access can appear on the left of simple or
compound assignment:

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
cache?[key] = value;
```

The right-hand side is evaluated only when the receiver is non-null. This is
important when it has side effects or is expensive. Null-conditional `++` and
`--` are not supported; express those updates another way.

## Visual Basic Runtime-API Compatibility

The Visual Basic compiler understands and enforces `unmanaged` generic
constraints exposed by runtime APIs. It also respects
`OverloadResolutionPriorityAttribute`, including for Span-oriented overloads.
This removes ambiguities and aligns overload selection with newer runtime API
designs. When a shared library exposes prioritized or span overloads, test its
Visual Basic callers rather than adding workarounds for the previous compiler
behavior.
