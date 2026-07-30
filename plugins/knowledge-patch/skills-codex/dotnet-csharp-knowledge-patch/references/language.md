# C# Language and Compiler Features

Use this reference for language syntax, overload-resolution changes, partial
members, preview type-system work, control flow, and pointer safety. Guidance
from `10.0-guides` describes the C# 14 language surface; preview material is
attributed to `csharp-15-preview`.

## Extension members

C# 14 extension blocks can add instance properties and methods, static
properties and methods, and operators to an extended type. A block with a
named receiver defines instance members:

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

Omit the receiver name when defining static extension members. An extension
indexer is an instance member and therefore requires a named receiver:

```csharp
public static class SequenceIndexers
{
    extension(IEnumerable<int> sequence)
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}
```

Extension indexers are part of `csharp-15-preview`.

## Properties, partial members, and assignments

### Field-backed properties

The contextual `field` token refers to the compiler-synthesized backing field,
so accessors can add behavior without declaring a field:

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

When the type already declares an identifier named `field`, use `@field` or
`this.field` to refer to the existing identifier.

### Partial constructors and events

Instance constructors and events can be partial. Each has exactly one defining
declaration and one implementing declaration. Only the implementing
constructor can specify `this()` or `base()`. The implementing declaration of
a partial field-like event supplies its `add` and `remove` accessors.

### User-defined compound assignment

A type can define a dedicated compound-assignment operator instead of relying
only on the corresponding binary operator. Use it when `x op= y` needs
behavior or efficiency distinct from `x = x op y`.

### Null-conditional assignment

`?.` and `?[]` can appear on the left of simple or compound assignments. The
right side is evaluated only when the receiver is non-null:

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

Null-conditional `++` and `--` remain unsupported.

## Names, lambdas, and spans

### Unbound generic types in `nameof`

An unbound generic type is valid in `nameof`; `nameof(List<>)` evaluates to
`"List"` without a type argument.

### Modifiers on implicitly typed lambda parameters

Simple lambda parameters can use `scoped`, `ref`, `in`, `out`, or
`ref readonly` without explicit parameter types:

```csharp
TryParse<int> parse =
    (text, out result) => int.TryParse(text, out result);
```

`params` still requires an explicitly typed parameter list.

### First-class span conversions

C# 14 adds implicit conversions among arrays, `Span<T>`, and
`ReadOnlySpan<T>`. Span types participate more naturally as extension
receivers, in composed conversions, and in generic type inference. Re-test
overload-sensitive code because span-aware overload resolution can select a
different overload after the language upgrade.

The Visual Basic compiler also understands `unmanaged` generic constraints and
honors `OverloadResolutionPriorityAttribute`, enabling Span-oriented overload
selection and resolving ambiguities consistently with the runtime API surface
(batch `10.0`).

## Collection construction

In `csharp-15-preview`, a first-position `with(...)` element in a collection
expression passes arguments to the underlying constructor or factory:

```csharp
List<string> names = [with(capacity: 8), "Ada", "Grace"];
HashSet<string> set =
    [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO"];
```

Use this for choices such as initial capacity or an equality comparer.

## Unions

The `union` keyword declares a value whose cases are listed types. Each case
converts implicitly to the union, and a `switch` expression that covers every
case is exhaustive:

```csharp
public record Cat(string Name);
public record Dog(string Name);
public union Pet(Cat, Dog);

string Name(Pet pet) => pet switch
{
    Cat cat => cat.Name,
    Dog dog => dog.Name,
};
```

In `csharp-15-preview`, the supporting `UnionAttribute` and `IUnion` runtime
types arrive with .NET 11 Preview 5. Some proposed union features remain
unimplemented; do not assume the full proposal is available.

## Closed hierarchies

The contextual `closed` modifier makes a class implicitly abstract and
restricts its direct descendants to its declaring assembly. This closed set
allows exhaustive matching:

```csharp
public closed record class GateState;
public record class Shut : GateState;
public record class Open(float Percent) : GateState;

string Describe(GateState state) => state switch
{
    Shut => "shut",
    Open(var percent) => $"{percent}% open",
};
```

The restriction is not transitive. In Preview 5, every project using the
feature must declare `System.Runtime.CompilerServices.ClosedAttribute` until
the runtime provides it.

## Labeled loop control

`break label;` exits a labeled enclosing loop or `switch`.
`continue label;` starts the next iteration of a labeled enclosing loop. Put
the label directly on its target statement:

```csharp
outer: for (int row = 0; row < height; row++)
{
    for (int column = 0; column < width; column++)
    {
        if (blocked[row, column])
            continue outer;
        if (goal[row, column])
            break outer;
    }
}
```

## Pointer safety

With the preview language version, these operations no longer require an
`unsafe` context:

- pointer declarations;
- address-of;
- `fixed`;
- pointer-targeted `stackalloc`; and
- `sizeof` on unmanaged types.

Dereferencing, pointer member or element access, and function-pointer
invocation still require `unsafe`:

```csharp
int value = 42;
int* pointer = &value;

unsafe
{
    Console.WriteLine(*pointer);
}
```

Requires-unsafe member propagation, assembly opt-in, and the `safe` keyword
were still planned for a later preview in `csharp-15-preview`.

In `11.0-preview.6`, pointer-taking APIs such as `Buffer.MemoryCopy`,
pointer-based `Span<T>` constructors, `Unsafe`, `NativeMemory`, and encoding
pointer overloads no longer carry `RequiresUnsafe`. They do not independently
require project-wide `AllowUnsafeBlocks`; the pointer operation itself still
determines whether an unsafe context is needed.
