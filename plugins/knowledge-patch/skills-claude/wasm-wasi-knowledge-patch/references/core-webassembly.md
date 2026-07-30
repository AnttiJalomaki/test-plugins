# Core WebAssembly Standard and Semantics

## Evergreen standard

WebAssembly 2.0 reached W3C Candidate Recommendation in December 2024 after the
language specification was completed in early 2022. Starting with 2.0, the
Candidate Recommendation is updated in place as an evergreen standard; the
GitHub-hosted specification carries the latest fixes and formats. These details
are attributed to `wasm-2.0`.

## Features standardized by 2.0

The backward-compatible 2.0 release standardized:

- 128-bit SIMD;
- bulk memory and table operations;
- multi-value results and block inputs;
- first-class function and external references;
- multiple typed tables;
- non-trapping float-to-integer conversions;
- sign-extension instructions.

## Live 3.0 feature set

WebAssembly 3.0 became the live standard with these completed features:

- memory64;
- multi-memory;
- garbage collection;
- typed references;
- tail calls;
- native exception handling;
- relaxed SIMD;
- a deterministic profile;
- text annotations.

The JavaScript embedding also gained string builtins. This section and the
remaining sections are attributed to `wasm-3.0`.

## 64-bit memories and tables

Memories and tables may use `i64` instead of `i32` as their address type. This
raises the theoretical address space from 4 GiB to 16 EiB. The Web embedding
still caps a 64-bit memory at 16 GiB.

Address width is therefore distinct from the allocation limit of a particular
embedding.

## Multiple memories

A module may define or import multiple memories and address each directly. It
can also copy data between memories.

This removes the former single-memory limitation for module-merging tools and
allows applications to use deliberately separate address spaces.

## Native exception dispatch

Exception tags declare their payload data. Handler blocks use dispatch lists
made of tag/label pairs, or catch-all labels, to choose where execution
continues after a throw.

Exception handling is consequently portable within Wasm and does not require a
throw to escape into the host.

## Deterministic execution profile

The deterministic profile fixes behavior for operations whose base semantics
permit more than one result. It currently covers:

- floating-point NaN generation;
- relaxed SIMD edge cases.

Reproducibility holds between platforms that elect to implement this profile.
Ordinary relaxed SIMD implementations may still choose any outcome permitted
by its base semantics.

## Custom text annotations

The text format supports generic annotations that tools may optionally ignore.
They are analogous to binary custom sections.

Core Wasm assigns these annotations no meaning. Downstream standards can define
specific annotation semantics without changing the core format.

## JavaScript string builtins

The JavaScript API provides an importable primitive library for direct access
to and manipulation of JavaScript strings passed to Wasm as external
references.
