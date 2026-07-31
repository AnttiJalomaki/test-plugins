# C++, Swift Interop, and Language Runtime Changes

## Enable hvf availability checking explicitly

Availability checking for hvf C APIs is disabled unless
`BUILD_FOR_APPLE_SDK` is defined before every hvf header include. Define it at
the top of the translation unit or in a build-wide prefix that is guaranteed
to precede those headers: (18.4)

```c
#define BUILD_FOR_APPLE_SDK 1
```

## Replace libxml2 custom allocators

The libxml2 custom-allocation API is deprecated on iOS 18.4. Replace
`xmlMalloc()` and `xmlMallocAtomic()` with `malloc()`, `xmlRealloc()` with
`realloc()`, `xmlFree()` with `free()`, and `xmlMemStrdup()` with `strdup()`.
(18.4)

Stop configuring allocation through `xmlMemSetup()`, `xmlMemGet()`,
`xmlGcMemSetup()`, `xmlGcMemGet()`, or their corresponding global variables.
libxml2 and libxslt now allocate internally with the system allocator.

## Use the public fileport calls

`fileport_makeport(2)` and `fileport_makefd(2)` are public APIs and have manual
pages. Code that needs Mach fileport conversion can use these public entry
points rather than private declarations. (18.4)

## Treat generic `std::char_traits` as temporary compatibility

Xcode 16.4 restores the generic `std::char_traits` base template that Xcode
16.3 removed. Nonstandard instantiations such as
`std::basic_string<long long>` compile again, but the base template remains
deprecated and is planned for removal. Migrate such code instead of treating
the restored template as a permanent API. (18.5)

## Audit libc++ container ABI boundaries

Xcode 26 can change the layout of `std::unordered_map`,
`std::unordered_multimap`, `std::unordered_set`, `std::unordered_multiset`, and
`std::deque` when an empty allocator shares a base across rebound allocator
types. (26.0)

An enclosing type's layout can also change when it contains a standard
container and the same empty allocator, comparator, or hasher appears as a
`[[no_unique_address]]` member or empty base. Audit values passed across dylib,
framework, plugin, IPC, or persisted binary boundaries; rebuild all sides with
a compatible toolchain where possible.

## Use updated Swift and C++ interop behavior

`MutableSpan` can be passed as an `inout` function parameter without enabling
an experimental feature. Remove obsolete feature gates that existed only for
this use. (26.0)

Swift infers `SWIFT_SHARED_REFERENCE` for a C++ derived type when its base type
already carries the annotation. Avoid duplicating the annotation solely to
make inheritance visible to Swift. (26.0)

## Diagnose Objective-C nonatomic-property races

During a synthesized nonatomic property mutation, the setter may briefly store
the sentinel `0x400000000000bad0`; on 32-bit watchOS the sentinel is `0xbad0`.
A concurrent reader that crashes on this value has exposed unsafe concurrent
access. Synchronize access or make the ownership design race-free rather than
filtering or retrying the sentinel value. (26.0)
