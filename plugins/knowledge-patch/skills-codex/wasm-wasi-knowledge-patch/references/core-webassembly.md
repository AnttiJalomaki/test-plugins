# Core WebAssembly standards and features

## Evergreen specification

WebAssembly 2.0 reached W3C Candidate Recommendation in December 2024 after
the language specification was completed in early 2022 (wasm-2.0). From that
release onward, the Candidate Recommendation is updated in place as an
evergreen standard. Treat the GitHub-hosted specification as the location of
the latest fixes and formats.

WebAssembly 3.0 became the live standard with its release on 2025-09-17
(wasm-3.0).

## Standardized feature sets

WebAssembly 2.0 is fully backward compatible and standardizes
(wasm-2.0):

- 128-bit SIMD;
- bulk memory and table operations;
- multi-value results and block inputs;
- first-class function and external references with multiple typed tables;
- non-trapping float-to-integer conversions; and
- sign-extension instructions.

WebAssembly 3.0 completes and standardizes (wasm-3.0):

- memory64 and multi-memory;
- garbage collection and typed references;
- tail calls and native exception handling;
- relaxed SIMD and a deterministic profile; and
- text annotations.

The JavaScript embedding also gains string builtins.

## 64-bit memories and tables

A memory or table can use `i64` instead of `i32` for its address type
(wasm-3.0). This raises the theoretical address space from 4 GiB to 16 EiB.
Web embeddings impose a smaller limit: a 64-bit memory is capped at 16 GiB.

Keep the theoretical core limit distinct from the Web embedding limit when
validating layouts or deciding whether a deployment target can instantiate a
module.

## Multiple memories

A module can define or import multiple memories and address them directly,
including copying data between memories (wasm-3.0). This removes the former
single-memory constraint from module-merging tools and permits intentionally
separate address spaces.

## Native exception dispatch

Exception tags declare their payload data (wasm-3.0). A handler block uses a
dispatch list containing tag/label pairs or catch-all labels to choose where
execution continues after a throw.

Use these semantics for portable exception handling inside Wasm rather than
forcing every exception to escape into the host.

## Deterministic execution profile

The deterministic profile defines one behavior for instructions whose base
semantics allow more than one result (wasm-3.0). The affected cases are
currently:

- floating-point NaN generation; and
- relaxed SIMD edge cases.

Reproducibility is a property of platforms that opt into this profile.
Ordinary relaxed SIMD may still choose any result permitted by its specified
semantics.

## Text annotations

The text format supports generic annotations that tools may ignore
(wasm-3.0). They are analogous to custom sections in the binary format. Core
Wasm assigns them no meaning; downstream standards can define concrete
annotations.

Do not assume an unknown annotation changes core execution semantics, and do
not reject it merely because the current tool does not interpret it.

## JavaScript string builtins

The JavaScript API exposes an importable primitive library for strings
(wasm-3.0). Wasm can use it to access and manipulate JavaScript strings passed
as external references without first translating them through a separate
linear-memory representation.
