# Language, Runtime, and Interoperability

Use this reference for C and Objective-C APIs, allocator changes, Swift compiler
behavior, Swift/C++ interoperation, and binary-layout risk.

## Contents

- [hvf Availability Checking](#hvf-availability-checking)
- [libxml2 System Allocation](#libxml2-system-allocation)
- [Public Fileport System Calls](#public-fileport-system-calls)
- [Generic `std::char_traits` Compatibility](#generic-stdchar_traits-compatibility)
- [Objective-C Nonatomic Property Race Sentinel](#objective-c-nonatomic-property-race-sentinel)
- [Swift Explicit Modules](#swift-explicit-modules)
- [`MutableSpan` as an `inout` Parameter](#mutablespan-as-an-inout-parameter)
- [Inherited C++ Shared-Reference Annotations](#inherited-c-shared-reference-annotations)
- [C++ Standard-Library ABI Edge Cases](#c-standard-library-abi-edge-cases)

## hvf Availability Checking

Availability checking for hvf C APIs is disabled unless
`BUILD_FOR_APPLE_SDK` is defined before any hvf header is included. For the iOS
18.4 SDK (`18.4`), place the definition before imports:

```c
#define BUILD_FOR_APPLE_SDK 1
```

Defining it after a header has already been processed does not retroactively
enable the checks.

## libxml2 System Allocation

The libxml2 custom-allocation API is deprecated on iOS 18.4 (`18.4`). Replace:

| Deprecated libxml2 entry point | System replacement |
| --- | --- |
| `xmlMalloc()`, `xmlMallocAtomic()` | `malloc()` |
| `xmlRealloc()` | `realloc()` |
| `xmlFree()` | `free()` |
| `xmlMemStrdup()` | `strdup()` |

Stop configuring allocators through `xmlMemSetup()`, `xmlMemGet()`,
`xmlGcMemSetup()`, `xmlGcMemGet()`, or their corresponding global variables.
libxml2 and libxslt now use the system allocator internally.

## Public Fileport System Calls

`fileport_makeport(2)` and `fileport_makefd(2)` are public APIs in iOS 18.4
(`18.4`) and have manual pages. Prefer these public declarations over private or
locally reproduced interfaces.

## Generic `std::char_traits` Compatibility

Xcode 16.4 (`18.5`) restores the generic base `std::char_traits` template that
Xcode 16.3 removed. Nonstandard instantiations such as
`std::basic_string<long long>` compile again.

This restoration is temporary compatibility. The generic base remains
deprecated and is planned for removal, so migrate nonstandard instantiations
rather than treating restored compilation as a durable contract.

## Objective-C Nonatomic Property Race Sentinel

With the iOS 26 SDK (`26.0`), a synthesized setter can briefly store
`0x400000000000bad0` during a nonatomic property mutation. On 32-bit watchOS the
sentinel is `0xbad0`.

If a concurrent reader crashes after observing this value, diagnose unsafe
concurrent access to the nonatomic property. Add correct synchronization or
isolation; do not special-case the sentinel as application data.

## Swift Explicit Modules

Xcode 26 (`26.0`) enables Swift explicit modules by default for Swift targets.
The default does not apply to targets using a pre-Swift-5 language version or
Swift/C++ interoperability.

For a severe compatibility problem, opt out temporarily:

```text
SWIFT_ENABLE_EXPLICIT_MODULES=NO
```

Keep the workaround scoped to affected targets and re-evaluate it as compiler
and dependency issues are resolved.

## `MutableSpan` as an `inout` Parameter

With Swift 6.2 in Xcode 26 (`26.0`), `MutableSpan` can be passed as an `inout`
function parameter without enabling an experimental feature. Remove obsolete
feature gates used solely for this capability.

## Inherited C++ Shared-Reference Annotations

Swift in Xcode 26 (`26.0`) infers `SWIFT_SHARED_REFERENCE` on a C++ type when
its base type already carries that annotation. Account for the inherited
semantics when reviewing imported ownership behavior; a repeated annotation on
every derived type is not required.

## C++ Standard-Library ABI Edge Cases

Xcode 26 (`26.0`) can change the layout of these C++ standard-library
containers when an empty allocator shares a base across rebound allocator
types:

- `std::unordered_map` and `std::unordered_multimap`
- `std::unordered_set` and `std::unordered_multiset`
- `std::deque`

An enclosing type's layout can also change when it contains one of those
containers and the same empty allocator, comparator, or hasher as either a
`[[no_unique_address]]` member or an empty base.

Rebuild all sides of a binary boundary together where possible. Audit persisted
raw layouts, IPC payloads, exported C++ ABIs, and mixed-toolchain dependencies
before assuming layout compatibility.
