# Targets and Offload

Target feature names, defaults, ABI layout, runtime requirements, and lowering
support can change independently. Validate the emitted target and deployed
runtime, not merely driver acceptance of a flag.

## Target defaults and ABI details

### Arm and AArch64

On 32-bit Arm, Clang 20 passes an empty C++ structure as a one-byte object to
match AAPCS32 and GCC; `-fclang-abi-compat=19` restores the old ignored-argument
behavior (`clang-20.1`). `-fno-omit-frame-pointer` now retains frame pointers in
leaf functions unless combined with `-momit-leaf-frame-pointer`. SME
function-type attributes participate in mangling.

In Clang 21, AArch32 `-mtp` defaults to `auto`, choosing `TPIDRURO` where
available instead of calling `__aeabi_read_tp`; use `-mtp=soft` if the call is
required (`clang-21.1`). The Arm assembler includes FPU features implied by the
selected CPU or architecture, so remove them with explicit `+no...` options.
`+nosimd` now disables NEON and dependent features. AArch64 adds
`-mexecute-only`/`-mpure-code` and `-msve-streaming-vector-bits=`. Replace
deprecated pointer-authentication `__has_feature` tests with `__PTRAUTH__`.

GCC 16 deprecates AArch64 PC-relative literal loads; migrate assembly and
low-level emitters rather than relying on future acceptance (`gcc-16.1`).

Clang 22 changes AArch64 argument passing for empty C++ classes with large
explicit alignment (`clang-22.1`). ACLE function multiversioning reaches release
status with PAC/BTI-aware resolvers, overridable version priority, and
unreachable-version diagnostics.

### X86

Clang 20 adds AVX10.2, MOVRS, AMX-FP8, AMX-TRANSPOSE, AMX-MOVRS,
AMX-AVX512, AMX-TF32, and `-march/-mtune=diamondrapids` (`clang-20.1`). MMX
header intrinsics using `__m64` now always require SSE2/XMM; inline MMX assembly
remains supported.

Clang 21 makes `-mavx10.1` select a 512-bit maximum vector width because the
AVX10/256 specification was removed (`clang-21.1`). The
`-mavx10.x-256`, `-mavx10.x-512`, and `-m[no-]evex512` forms warn; use
`-m[no-]avx10.x`.

Clang 22 removes `-mavx10.x-{256,512}`, `-mno-avx10.x-{256,512}`, and the
EVEX512 forms; intrinsic requests use unsuffixed `avx10.x` (`clang-22.1`). It adds `-march=wildcatlake` and
`-march=novalake`. clang-cl adds `/arch:AVX10.1`, `/arch:AVX10.2`, `/vlen`,
`/vlen=256`, and `/vlen=512`; more SSE/AVX/AVX512 intrinsics are constant-
expression capable.

### Other target changes

Clang 20 adds gfx950, Arm SVE2.1/SME2.1, AArch64 `fujitsu-monaka`, RISC-V
`-mcmodel=large`, RVV intrinsics 1.0, CUDA SDK 12.6, and `sm_100`
(`clang-20.1`). `target_version` is limited to AArch64 and RISC-V; on AArch64,
`target_version("default")` by itself creates a mangled default version.

Clang 21 (`clang-21.1`) adds Cortex-A320, MIPS little-endian Windows, OHOS and
`_Float16`/`__bf16` on LoongArch, RISC-V `-mtune=generic-ooo`, SiFive and
Qualcomm interrupt attributes, and `__builtin_riscv_pause()`. AIX runtimes moved
from `lib/clang/20/lib/aix` to per-target Clang 21 directories.

Also in Clang 21:

- AMDGPU defaults to code object version 6 and requires ROCm 6.3 at runtime.
- Hexagon defaults to V68 instead of V60.
- LoongArch `_BitInt(N)` wider than 64 bits has consistent 16-byte alignment,
  potentially changing layout.

Clang 22 enables linker relaxation by default on LoongArch64 and supports
LoongArch32 (`clang-22.1`). RISC-V adds `-march=unset`, falling back to `-mcpu`
or platform defaults, and defines `__GCC_CONSTRUCTIVE_SIZE` and
`__GCC_DESTRUCTIVE_SIZE` as 64. `wasm32-wasi` is deprecated in favor of
`wasm32-wasip1`.

## Driver and feature detection

Clang 20's WebAssembly `generic` CPU enables bulk memory and non-trapping
float-to-int conversion (`clang-20.1`). clang-cl adds `/std:c++23preview`, and
COFF targets support `#pragma clang section`.

During Clang 22 offloading, `__has_builtin` considers only the currently active
target (`clang-22.1`). OpenCL no longer exposes unconditional header-only feature
macros; availability is centralized and controlled consistently through
`-cl-ext`.

## CUDA and HIP

Clang 20 uses the new CUDA offloading driver by default, including native
`-fgpu-rdc` static-library support (`clang-20.1`). Its RDC binary format is not
compatible with NVIDIA's. Use `--no-offload-new-driver` for the old path during
migration.

Clang 22 supports device-side C++17 CTAD in CUDA and HIP (`clang-22.1`).
Deduction guides act as implicit `__host__ __device__` declarations; duplicate
implicit guides are suppressed and constraint-distinct guides remain. Explicit
target-only guides are errors. Explicit host-plus-device guides work but are
deprecated because deduction guides do not generate code.

## OpenMP

Clang 20 adds (`clang-20.1`):

- `omp assume` and `omp scope`;
- allocator and alignment modifiers on `allocate`;
- combined masked-taskloop forms, optionally with `parallel` and `simd`.

DeviceRTL uses generic IR. `LIBOMPTARGET_DEVICE_ARCHITECTURES` is unused, and
runtime builds always cover AMDGPU and NVPTX.

Clang 21 adds the `no_openmp_constructs` assumption, `self_maps` in map and
requirement clauses, `omp stripe`, and private-variable reduction
(`clang-21.1`). The delimited `declare target` form is deprecated.

Clang 22 adds (`clang-22.1`):

- `need_device_addr` for `adjust_args`, `threadset`, `groupprivate`, `omp fuse`,
  omitted array-section lengths, and the new `uses_allocators` syntax;
- `variable-category`, `defaultmap(storage|private)`, and `default` on `target`;
- the optional OpenMP 6.0 `nowait` argument;
- OpenMP 6.1 `fb_nullify` and `fb_preserve` fallbacks for `need_device_ptr`.

If lookup fails, `use_device_ptr` and `use_device_addr` now preserve the host
address.

## OpenACC

Clang 21's `-fopenacc` performs OpenACC 3.4 semantic analysis and builds an AST
(`clang-21.1`). Partial lowering exists only in a Clang-IR-enabled compiler with
`-fclangir`. The ACC MLIR dialect cannot lower to LLVM IR, so OpenACC code
generation is not yet available.
