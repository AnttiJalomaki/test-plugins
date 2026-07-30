# Targets, Offloading, and OpenMP

Treat target defaults, ABI rules, device-toolchain formats, and feature probes
as part of the build contract. Clean host and device artifacts together after
changing any of them.

## Removed and deprecated targets

Clang removed the `le32` and `le64` targets, RenderScript target support, and
the `clang-rename` tool in clang-20.1. On SPARC Linux, `clang -m32` defaults to
`-mcpu=v9`; distributions retaining SPARC V8 must pass `-mcpu=v8`.

GCC 15 removed Nios II and Solaris 11.3 support and deprecated AArch64 ILP32
(`-mabi=ilp32`). It is the final GCC release with the old `reload` register
allocator, so targets without LRA support are affected by its GCC 16 removal
(gcc-15.1).

GCC 16 deprecates AArch64 PC-relative literal loads. Assembly and low-level code
that emits them should migrate rather than depending on continued acceptance
(gcc-16.1).

## X86 feature selection

### MMX header intrinsics (clang-20.1)

The `*mmintrin.h` intrinsics on `__m64` use SSE2 and XMM registers. They no
longer work for MMX-only targets or `-mmmx -mno-sse2`. MMX inline assembly
remains supported; direct users of removed `__builtin_ia32_*` implementation
builtins must use the header intrinsics.

### AVX10 selection (clang-21.1)

`-mavx10.1` selects a 512-bit maximum vector width because AVX10/256 was
removed from the specification. The `-mavx10.x-256`,
`-mavx10.x-512`, and `-m[no-]evex512` spellings warn; use
`-m[no-]avx10.x`.

### Removed spellings and clang-cl controls (clang-22.1)

The deprecated `-mavx10.x-{256,512}`, `-mno-avx10.x-{256,512}`, and
`-m[no-]evex512` spellings were removed. Intrinsic feature requests use
unsuffixed `avx10.x`.

Clang adds `-march=wildcatlake` and `-march=novalake`. clang-cl adds
`/arch:AVX10.1`, `/arch:AVX10.2`, `/vlen`, `/vlen=256`, and `/vlen=512`.
More SSE, AVX, and AVX512 intrinsics are usable in constant expressions.

## Arm and AArch64

On 32-bit Arm, empty C++ structures are passed as one-byte objects in
clang-20.1; use `-fclang-abi-compat=19` for the older ignored-argument ABI.
`-fno-omit-frame-pointer` now retains leaf frame pointers unless
`-momit-leaf-frame-pointer` is also passed. SME function-type attributes
participate in mangling.

The Arm assembler in clang-21.1 includes FPU features implied by the selected
CPU or architecture. Remove them with explicit `+no...` options; `+nosimd`
actually disables NEON and dependent features. AArch64 adds
`-mexecute-only`/`-mpure-code` and `-msve-streaming-vector-bits=`.
Replace deprecated pointer-authentication `__has_feature` checks with
`__PTRAUTH__`.

In clang-22.1, AArch64 argument passing changes for empty C++ classes with large
explicit alignment. ACLE function multiversioning reaches release status with
PAC/BTI-aware resolvers, overridable version priority, and unreachable-version
diagnostics.

## Architecture additions and defaults

### clang-20.1

New support includes gfx950, AVX10.2, MOVRS,
AMX-FP8/TRANSPOSE/MOVRS/AVX512/TF32,
`-march/-mtune=diamondrapids`, Arm SVE2.1/SME2.1,
AArch64 `fujitsu-monaka`, RISC-V `-mcmodel=large` and RVV intrinsics 1.0,
CUDA SDK 12.6, and `sm_100`.

`target_version` is limited to AArch64 and RISC-V.
`target_version("default")` alone creates a mangled AArch64 default function
version.

### clang-21.1

New support includes Cortex-A320, MIPS little-endian Windows targets, OHOS and
`_Float16`/`__bf16` on LoongArch, RISC-V `-mtune=generic-ooo`, new SiFive and
Qualcomm interrupt attributes, and `__builtin_riscv_pause()`.

AIX compiler runtimes moved from `lib/clang/20/lib/aix` to per-target Clang 21
directories.

AMDGPU defaults to code object version 6, requiring ROCm 6.3 at run time.
Hexagon's default target moves from V60 to V68. LoongArch `_BitInt(N)` wider
than 64 bits has consistent 16-byte alignment, which can change ABI layout.

### clang-22.1

LoongArch64 enables linker relaxation by default, and LoongArch32 is supported.
RISC-V adds `-march=unset` to fall back to `-mcpu` or platform defaults and
sets `__GCC_CONSTRUCTIVE_SIZE` and `__GCC_DESTRUCTIVE_SIZE` to 64.
`wasm32-wasi` is deprecated in favor of `wasm32-wasip1`.

## Driver and offloading behavior

### CUDA, WebAssembly, clang-cl, and COFF (clang-20.1)

CUDA uses the new offloading driver by default and supports native
`-fgpu-rdc` static libraries. Its RDC binary format is incompatible with
NVIDIA's; `--no-offload-new-driver` restores the old path.

WebAssembly's `generic` CPU enables bulk memory and non-trapping float-to-int
conversion. clang-cl adds `/std:c++23preview`, and COFF targets add
`#pragma clang section`.

### Target-aware feature detection (clang-22.1)

During offloading, `__has_builtin` considers only the currently active target.
Probe host and device separately. OpenCL's formerly unconditional header-only
feature macros were removed; extension and feature availability is centralized
and controlled through `-cl-ext`.

### CUDA and HIP CTAD (clang-22.1)

C++17 deduction guides behave as implicit `__host__ __device__` declarations
for CUDA and HIP. Duplicate implicit guides are suppressed and
constraint-distinct guides are preserved.

Explicit target-only guides are errors. Explicit host-plus-device guides remain
accepted but are deprecated because deduction guides do not participate in code
generation.

## OpenACC

`-fopenacc` performs OpenACC 3.4 semantic analysis and AST construction in
clang-21.1. Partial lowering requires a Clang-IR-enabled compiler and
`-fclangir`. The ACC MLIR dialect cannot lower to LLVM IR, so OpenACC code
generation is not available.

## OpenMP language and runtime

### Allocation, scope, and DeviceRTL (clang-20.1)

Clang adds `omp assume`, `omp scope`, allocator and alignment modifiers on
`allocate`, and combined masked-taskloop forms with optional `parallel` and
`simd`.

DeviceRTL uses generic IR, so `LIBOMPTARGET_DEVICE_ARCHITECTURES` is unused and
builds always cover AMDGPU and NVPTX.

### Assumptions, mapping, and reductions (clang-21.1)

Clang adds the `no_openmp_constructs` assumption clause, `self_maps` in map and
requirement clauses, `omp stripe`, and private-variable reduction. The
delimited form of `declare target` is deprecated.

### OpenMP 6.x syntax and fallback behavior (clang-22.1)

Clang adds:

- `need_device_addr` for `adjust_args`;
- `threadset`, `groupprivate`, and `omp fuse`;
- omitted array-section lengths;
- the new `uses_allocators` syntax;
- `variable-category`;
- `defaultmap(storage|private)`; and
- `default` on `target`.

OpenMP 6.0 permits an optional `nowait` argument. OpenMP 6.1 adds `fb_nullify`
and `fb_preserve` fallbacks to `need_device_ptr`.
`use_device_ptr` and `use_device_addr` preserve host addresses when lookup
fails.
