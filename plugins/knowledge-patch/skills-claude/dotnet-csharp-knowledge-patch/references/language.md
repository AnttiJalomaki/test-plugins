# C# and language features

## Extension members and indexers

C# 14 extension blocks can add instance properties and methods, static
properties and methods, and operators to an extended type
(`10.0-guides`). A block with a named receiver defines instance members;
omitting the receiver name permits static members.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

The C# 15 preview adds extension indexers (`csharp-15-preview`). An indexer is
an instance member, so its extension block must have a named receiver.

```csharp
public static class SequenceIndexers
{
    extension(IEnumerable<int> sequence)
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}
```

## Properties, assignments, and partial members

### Field-backed properties

The contextual `field` token refers to a compiler-synthesized backing field,
allowing validation or transformation without declaring a field. In a type
that already declares an identifier named `field`, use `@field` or
`this.field` to refer to that identifier.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

### Null-conditional and compound assignment

`?.` and `?[]` can appear on the left of simple and compound assignments. The
right-hand expression is evaluated only when the receiver is non-null.
Increment and decrement remain unsupported in this form.

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

C# 14 also supports user-defined compound-assignment operators, so a type can
provide dedicated compound-assignment behavior instead of falling back only
to the corresponding binary operator.

### Partial constructors and events

Instance constructors and events can be partial. Each requires exactly one
defining declaration and one implementing declaration. Only the implementing
constructor may specify `this()` or `base()`. The implementing part of a
partial field-like event supplies its `add` and `remove` accessors.

## Conversions, names, and lambdas

### First-class span conversions

C# 14 adds implicit conversions among arrays, `Span<T>`, and
`ReadOnlySpan<T>`. Span types participate more naturally as extension
receivers, in composed conversions, and in generic type inference. This also
affects overload resolution: after changing language versions, a span-aware
overload can be selected where a different overload previously won.

### Unbound generic types in `nameof`

`nameof` accepts an unbound generic type:

```csharp
string name = nameof(List<>); // "List"
```

No type argument is required.

### Modifiers on implicitly typed lambda parameters

Simple lambda parameters can use `scoped`, `ref`, `in`, `out`, or
`ref readonly` without explicit parameter types. `params` still requires an
explicitly typed parameter list.

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

## Collection expression arguments

In the C# 15 preview, a `with(...)` element in the first position of a
collection expression supplies arguments to the underlying constructor or
factory. It can select initial capacity, equality comparers, or other
supported creation parameters.

```csharp
List<string> names = [with(capacity: 8), "Ada", "Grace"];
HashSet<string> set =
    [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO"];
```

## Union types

The `union` keyword declares a value whose cases are the listed types. Each
case converts implicitly to the union, and a `switch` expression that covers
every case is exhaustive.

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

The supporting `UnionAttribute` and `IUnion` runtime types arrive with .NET 11
Preview 5. Some proposed union capabilities are still unimplemented, so keep
usage within syntax and APIs available in the selected preview.

## Closed hierarchies

The contextual `closed` modifier restricts direct descendants of an
implicitly abstract class to its declaring assembly. This enables exhaustive
matching over those direct descendants, but the restriction is not
transitive.

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

In Preview 5, each project must declare
`System.Runtime.CompilerServices.ClosedAttribute` until the runtime supplies
it.

## Labeled control flow

`break label;` exits a labeled enclosing loop or `switch`.
`continue label;` starts the next iteration of a labeled enclosing loop. Put
the label directly on the target statement.

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

## Pointer safety boundaries

With the preview language version, pointer declarations, address-of, `fixed`,
pointer-targeted `stackalloc`, and `sizeof` on unmanaged types no longer
require an `unsafe` context.

```csharp
int value = 42;
int* pointer = &value;

unsafe
{
    Console.WriteLine(*pointer);
}
```

Dereferencing, pointer member or element access, and function-pointer
invocation still require an `unsafe` context. Requires-unsafe member
propagation, assembly opt-in, and the `safe` keyword are planned for a later
preview and should not be assumed available.

At the runtime API level, pointer-taking APIs such as `Buffer.MemoryCopy`,
pointer-based `Span<T>` constructors, `Unsafe`, `NativeMemory`, and encoding
pointer overloads no longer carry `RequiresUnsafe` in .NET 11 Preview 6
(`11.0-preview.6`). They therefore do not independently require project-wide
`AllowUnsafeBlocks` beyond whatever unsafe context the pointer operation
itself requires.

## Visual Basic compiler compatibility

The .NET 10 Visual Basic compiler understands and enforces `unmanaged` generic
constraints and respects `OverloadResolutionPriorityAttribute` (`10.0`).
This enables span-oriented overload selection and resolves ambiguities in the
same way as newer runtime APIs.
