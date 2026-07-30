# C++, Swift Interop, and Language Runtime Changes

## libc++ compatibility

### Generic char traits

Xcode 16.4 restores the generic `std::char_traits` base template that Xcode
16.3 removed, so nonstandard types such as `std::basic_string<long long>`
compile again. This is temporary compatibility: the primary template remains
deprecated and is planned for removal. Move custom character types to a proper
specialization rather than depending on the restored fallback (18.5).

### Associative lookup

`std::multimap::find` and `std::multiset::find` no longer necessarily return
the first equivalent item. Use `lower_bound` or `equal_range` when order among
equivalent elements matters. Bounds from `std::map` and `std::set` can also
change when the comparator is not a strict weak order; fix the comparator.
`_LIBCPP_ENABLE_LEGACY_TREE_LOWER_UPPER_BOUND` is a temporary compatibility
escape hatch (27.0-beta4).

## Standard-library ABI

Xcode 26 can change the layout of `std::unordered_map`,
`std::unordered_multimap`, `std::unordered_set`, `std::unordered_multiset`, and
`std::deque` when an empty allocator shares a base across rebound allocator
types. An enclosing type can also change layout when it contains a standard
container and the same empty allocator, comparator, or hasher as a
`[[no_unique_address]]` member or empty base. Rebuild both sides of an ABI
boundary and do not exchange affected layouts between mismatched toolchains
(26.0).

## Safe wrappers and spans

Safe-wrapper generation accepts a direct `std::span<T>` parameter annotated
`__noescape`. A different template instantiation elsewhere in the same
signature still blocks wrapper generation unless it is hidden behind a typedef
(27.0-beta4):

```cpp
void bar(std::span<int> y __noescape);
```

`MutableSpan` can be passed as an `inout` function parameter without enabling
an experimental feature (26.0).

Swift infers `SWIFT_SHARED_REFERENCE` for a C++ type whose base type already
has that annotation (26.0).

## Objective-C race diagnostic

A synthesized nonatomic-property setter may briefly store
`0x400000000000bad0`, or `0xbad0` on 32-bit watchOS, while mutating the
property. A concurrent reader that crashes on this sentinel has exposed unsafe
concurrent access; synchronize the property rather than treating the value as
ordinary corruption (26.0).
