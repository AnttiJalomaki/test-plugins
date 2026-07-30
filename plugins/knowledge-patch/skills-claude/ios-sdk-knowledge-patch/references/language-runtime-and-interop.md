# Language, Runtime, and Interoperability

## Swift concurrency and memory safety

### Core Data imports

With the iOS 26 SDK, `NSManagedObject` imports as nonisolated and non-`Sendable`, `NSManagedObjectContext` as nonisolated and `Sendable`, and the `perform`/`performBlock` family with `Sendable` closures. Rebuilds can expose new concurrency warnings. Keep managed objects within their context's scope and diagnose violations with `-com.apple.CoreData.ConcurrencyDebug 1` (26.0).

### Preference callbacks

SwiftUI's `onPreferenceChange` closure no longer has to be `@Sendable`, avoiding needless diagnostics when the closure accesses main-actor-isolated state (18.4).

### MutableSpan

`MutableSpan` can be passed as an `inout` function parameter without enabling an experimental feature (26.0).

## Localization and formatting

Interpolating a nonlocalized type into `LocalizedStringResource`, `String(localized:)`, or `AttributedString(localized:)` now produces a deprecation warning. Supply a localized value or deliberately convert descriptive output with `String(describing:)` (26.0).

SwiftUI deprecates concatenating `Text` with `+`. Use `Text` interpolation so translations can reorder inserted content (26.0).

`ISO8601FormatStyle` accepts fractional seconds regardless of `includingFractionalSeconds` (26.0), and it accepts hours-only time-zone offsets (26.6-rc).

## Objective-C concurrency diagnostics

A synthesized nonatomic setter can briefly write the sentinel `0x400000000000bad0`, or `0xbad0` on 32-bit watchOS. If a concurrent reader crashes on that value, the failure identifies unsafe concurrent property access; synchronize the mutation and read rather than filtering the value (26.0).

## C allocation and system calls

### libxml2 allocation

The custom allocator API is deprecated on iOS 18.4. Replace `xmlMalloc()`, `xmlMallocAtomic()`, `xmlRealloc()`, `xmlFree()`, and `xmlMemStrdup()` with `malloc()`, `realloc()`, `free()`, and `strdup()` (18.4).

Stop configuring allocation through `xmlMemSetup()`, `xmlMemGet()`, `xmlGcMemSetup()`, `xmlGcMemGet()`, or the corresponding globals. libxml2 and libxslt now use the system allocator internally.

### Fileports

`fileport_makeport(2)` and `fileport_makefd(2)` are public APIs and have manual pages (18.4).

## C++ compatibility

### Generic char traits

Xcode 16.4 restores the generic `std::char_traits` base template that Xcode 16.3 removed, so nonstandard types such as `std::basic_string<long long>` compile again. The base template remains deprecated and is planned for removal; use the restoration only as migration time (18.5).

### Shared-reference annotations

Swift infers `SWIFT_SHARED_REFERENCE` for a C++ type when its base type already carries the annotation (26.0).

### Direct spans in safe wrappers

Safe-wrapper generation accepts a direct `std::span<T>` parameter annotated `__noescape` (27.0-beta4):

```cpp
void bar(std::span<int> y __noescape);
```

Another template instantiation elsewhere in the signature still blocks wrapper generation unless that instantiation is hidden behind a typedef.

### Standard-container ABI edges

Xcode 26 can change the layout of `std::unordered_map`, `std::unordered_set`, their multi variants, and `std::deque` when an empty allocator shares a base across rebound allocator types. An enclosing type can also change layout when it contains a standard container and the same empty allocator, comparator, or hasher as a `[[no_unique_address]]` member or empty base. Rebuild ABI boundaries together or redesign the shared empty types (26.0).

### Associative lookup semantics

`std::multimap::find` and `std::multiset::find` no longer necessarily return the first equivalent element. Use `lower_bound` or `equal_range` when equivalent-element order matters (27.0-beta4).

Bounds for `std::map` and `std::set` may change when their comparator is not a strict weak order. Correct the comparator. `_LIBCPP_ENABLE_LEGACY_TREE_LOWER_UPPER_BOUND` is a temporary migration escape hatch, not a permanent substitute.

## Swift System file status

Swift System provides a `Stat` type covering `stat`, `lstat`, `fstat`, and `fstatat`. Initialize it from a `FilePath`, `FileDescriptor`, or C string, or call `FilePath.stat()` and `FileDescriptor.stat()` for instance access (27.0-beta4).
