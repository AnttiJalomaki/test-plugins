# Kernel, Architecture, and Core APIs

Use this reference for Zephyr work in this topic area. Entries are organized by developer task rather than release order.

## Architecture current-pointer hooks (4.1.0)

Architecture ports can provide a custom current-thread implementation with `CONFIG_ARCH_HAS_CUSTOM_CURRENT_IMPL`. RISC-V can keep the current-thread pointer in the global pointer register with `CONFIG_RISCV_CURRENT_VIA_GP`.

## Architecture Kconfig changes (migration-4.2)

`CONFIG_SRAM_VECTOR_TABLE` now additionally depends on `CONFIG_XIP`, `CONFIG_ARCH_HAS_VECTOR_TABLE_RELOCATION`, and `CONFIG_ROMSTART_RELOCATION_ROM`. Rename the x86-only `CONFIG_DEBUG_INFO` option to `CONFIG_X86_DEBUG_INFO`.

## Architecture support and execution protection (4.2.0)

Zephyr gains initial Renesas RX support, including `rsk_rx130` and a QEMU-based target, while NIOS2 support is removed. With `CONFIG_ARM_MPU_PXN` and `CONFIG_USERSPACE`, `__ramfunc` and `__ram_text_reloc` are privileged-execute-never, so privileged code can no longer execute from those regions.

## Asynchronous runtime power management (4.2.0)

Device runtime PM can execute synchronously or asynchronously and can use the system workqueue or a dedicated workqueue. Configure this with `CONFIG_PM_DEVICE_RUNTIME_ASYNC`, `CONFIG_PM_DEVICE_RUNTIME_USE_SYSTEM_WQ`, or `CONFIG_PM_DEVICE_RUNTIME_USE_DEDICATED_WQ` and the dedicated-workqueue size, priority, and init-priority options.

## Cache coherence API (migration-4.4)

Rename `CONFIG_ARCH_HAS_COHERENCE` to `CONFIG_CACHE_CAN_SAY_MEM_COHERENCE` and replace `arch_mem_coherent()` with `sys_cache_is_mem_coherent()`. Rename `CONFIG_CACHE_DOUBLEMAP` to `CONFIG_CACHE_HAS_MIRRORED_MEMORY_REGIONS`.

## Compiler-assisted instrumentation (4.3.0)

`CONFIG_INSTRUMENTATION` adds runtime call-graph tracing and statistical profiling through compiler-managed function instrumentation. It provides call-graph and statistical mode settings, trigger/stop and exclusion controls, and `instr_*` APIs for control and UART dumps.

## Core API removals (4.0.0)

Replace `K_THREAD_STACK_MEMBER` with `K_KERNEL_STACK_MEMBER`, `ceiling_fraction` with `DIV_ROUND_UP`, the architecture CMSIS headers with `cmsis_core.h`, and `<zephyr/random/rand32.h>` with `<zephyr/random/random.h>`. `CBPRINTF_PACKAGE_COPY_*`, generated `_ENUM_TOKEN`/`_ENUM_UPPER_TOKEN`, deprecated `net_pkt` functions, and the `EARLY`, `APPLICATION`, and `SMP` device-init levels are gone; `net_buf_put()`/`net_buf_get()` and the kscan subsystem are deprecated.

## CPU load and frequency scaling (4.3.0)

The new `cpu_load` subsystem derives CPU-usage metrics from scheduler statistics. Experimental policy-driven dynamic clock scaling is selected with `CONFIG_CPU_FREQ` and can use those metrics to balance performance and power.

## Device initialization errors (migration-4.3)

`device_init()` now returns a negative `-errno` on initialization failure. Remove workarounds that interpreted the earlier erroneous positive value.

## File-descriptor table sizing (migration-4.3)

`ZVFS_OPEN_SIZE` now determines file-descriptor table size and availability, with subsystem requirements contributed by `CONFIG_ZVFS_OPEN_ADD_SIZE_*`. `CONFIG_ZVFS_OPEN_MAX` remains but is raised to larger contributed minima unless `CONFIG_ZVFS_OPEN_IGNORE_MIN` is enabled.

## Hardware shadow stacks and Intel CET (4.3.0)

Zephyr adds architecture and kernel hardware-shadow-stack support through `CONFIG_ARCH_HAS_HW_SHADOW_STACK`, `CONFIG_HW_SHADOW_STACK`, sizing/declaration macros, and `k_thread_hw_shadow_stack_attach()`. x86 Intel CET and indirect-branch tracking are selected through the `CONFIG_X86_CET*` options.

## In-memory core dumps (4.2.0)

`CONFIG_DEBUG_COREDUMP_BACKEND_IN_MEMORY` and `CONFIG_DEBUG_COREDUMP_BACKEND_IN_MEMORY_SIZE` retain a core dump in RAM. Minimal Cortex-M memory dumps now include the thread stack top by default through `CONFIG_DEBUG_COREDUMP_THREAD_STACK_TOP`.

## Kernel shell commands (migration-4.0)

The `kernel threads` and `kernel stacks` commands are now `kernel thread list` and `kernel thread stacks`.

## Loadable extensions and demand paging (4.0.0)

Devicetree devices are exported to LLEXT, and ARM64 gains initial LLEXT and demand-paging support. Demand paging also gains LRU eviction, SMP compatibility, and on-demand mappings through `CONFIG_DEMAND_MAPPING`.

## Maximum CPU count (migration-4.0)

`CONFIG_MP_NUM_CPUS` was removed. Use `CONFIG_MP_MAX_NUM_CPUS`.

## Other removed core interfaces (4.1.0)

Replace `CONFIG_PM_DEVICE_RUNTIME_EXCLUSIVE` with `CONFIG_PM_DEVICE_SYSTEM_MANAGED` and `z_arch_esf_t` with `struct arch_esf`. `z_pm_save_idle_exit()`, `CONFIG_WIFI_NM_WPA_SUPPLICANT_CRYPTO`, `CONFIG_NET_PKT_BUF_DATA_POOL_SIZE`, and `CONFIG_NET_TCP_ACK_TIMEOUT` are removed.

## POSIX and kernel shell behavior (4.0.0)

The POSIX surface adds device I/O, signals, synchronized I/O, priority protection, `O_TRUNC`, `rmdir()`, `remove()`, and the reentrant time functions. The kernel shell can change thread CPU affinity at runtime, and bare `kernel reboot` now performs a cold reboot.

## POSIX and RISC-V Kconfig deprecations (4.3.0)

Rename `CONFIG_POSIX_READER_WRITER_LOCKS` to `CONFIG_POSIX_RW_LOCKS` and RISC-V's `CONFIG_EXTRA_EXCEPTION_INFO` to `CONFIG_EXCEPTION_DEBUG`; Newlib can opt into POSIX limits with `CONFIG_NEWLIB_LIBC_USE_POSIX_LIMITS_H`.

## POSIX headers and limits (migration-4.3)

Applications must include `<time.h>`, `<signal.h>`, and `<limits.h>` rather than the former `<zephyr/posix/...>` headers; non-POSIX C library ports may use Zephyr's `posix_time.h`, `posix_signal.h`, and `posix_limits.h`. Runtime-dependent limits may need to be obtained with `sysconf()`.

## Pressure-based CPU frequency policy (4.4.0)

The CPU frequency subsystem can select `CONFIG_CPU_FREQ_POLICY_PRESSURE` to scale frequency from scheduler load pressure.

## Reworked pipe API (migration-4.1)

The `CONFIG_PIPES` API is deprecated; the replacement pipe API is enabled automatically with `CONFIG_MULTITHREADING`. Replace `k_pipe_put()`/`k_pipe_get()` with `k_pipe_write()`/`k_pipe_read()`: `min_xfer` is gone, the byte count is returned directly, and threshold-based partial transfers are no longer supported.

Replace both flush calls with nonblocking `k_pipe_reset()`. Dynamic allocation through `k_pipe_alloc_init()`/`k_pipe_cleanup()` and availability queries are removed; `k_pipe_close()` instead closes a pipe and wakes waiters with an error, while buffered data remains readable until empty and `k_pipe_init()` reopens it.

## RISC-V Devicetree ownership (migration-4.4)

`CONFIG_RISCV` now requires a `riscv` Devicetree node, whose `riscv,isa-base` and `riscv,isa-extensions` properties define the base ISA and extensions; `riscv,isa` is deprecated. SoC Kconfigs that encoded ISA/FPU choices, including the CV64A6 variants and AE350 `CONFIG_RV*`/FPU options, are removed or consolidated.

## RISC-V fatal exception frames (4.0.0)

With `CONFIG_EXTRA_EXCEPTION_INFO`, `arch_esf` now has a `csf` pointer to the callee-saved registers for use by `k_sys_fatal_error_handler()`. SoCs selecting `RISCV_SOC_HAS_ISR_STACKING` must include that member in `SOC_ISR_STACKING_ESF_DECLARE`.

## RISC-V machine timer binding (migration-4.2)

Several vendor machine-timer compatibles are unified as `riscv,machine-timer`. Both MTIME and MTIMECMP addresses must be explicit, with matching required names:

```devicetree
reg = <0xd1000000 0x8>, <0xd1000008 0x8>;
reg-names = "mtime", "mtimecmp";
```

The CPU group's `timebase-frequency` property can now supply `CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC`.

## RTIO callback chains (migration-4.3)

RTIO callback operations gain an argument containing the first error result in the chain. Callbacks now run even when an earlier submission failed, so handlers must inspect that result instead of assuming prior success.

## Runtime power-management defaults (4.3.0)

`CONFIG_PM_DEVICE_RUNTIME_DEFAULT_ENABLE` can enable device runtime power management by default, and drivers gain `pm_device_driver_deinit()` for deinitialization.

## Scheduler and I3C configuration (4.2.0)

Replace deprecated `CONFIG_SCHED_DUMB` and `CONFIG_WAITQ_DUMB` with `CONFIG_SCHED_SIMPLE` and `CONFIG_WAITQ_SIMPLE`. I3C group addressing and `CONFIG_I3C_USE_GROUP_ADDR` are removed; choose `CONFIG_I3C_CONTROLLER_ROLE_ONLY`, `CONFIG_I3C_TARGET_ROLE_ONLY`, or `CONFIG_I3C_DUAL_ROLE` through `CONFIG_I3C_MODE`.

## Scope-based cleanup (4.4.0)

`SCOPE_VAR_DEFINE`, `SCOPE_GUARD_DEFINE`, and `SCOPE_DEFER_DEFINE`, with the `scope_var`, `scope_guard`, and `scope_defer` helpers, provide RAII/defer-style cleanup when C scope exits.

## Stack-canary strength (migration-4.1)

`CONFIG_STACK_CANARIES` no longer adds `-fstack-protector-all`. Enable `CONFIG_STACK_CANARIES_ALL` when all-function stack protection is required.

## Suspend-to-RAM ownership (migration-4.3)

Applications must stop selecting `CONFIG_PM_S2RAM` and `PM_S2RAM_CUSTOM_MARKING`; SoCs and enabled `suspend-to-ram` devicetree power states now control them. Updated RW61x `exit-latency-us` values may also require increasing `min-residency-us` and can change power-state selection.

## System-timer low-power companion (migration-4.4)

Out-of-tree Cortex-M timer code should replace `z_cms_lptim_hook_on_lpm_entry/exit` with `z_sys_clock_lpm_enter/exit`, the `CONFIG_CORTEX_M_SYSTICK_LPM_TIMER_*` family with `CONFIG_SYSTEM_TIMER_LPM_COMPANION_*`, and `/chosen/zephyr,cortex-m-idle-timer` with `/chosen/zephyr,system-timer-companion`.

## Tiered heap hardening (4.4.0)

`CONFIG_SYS_HEAP_HARDENING` adds Basic, Moderate, Full, and Extreme checking for `sys_heap_alloc()` and `sys_heap_free()`, progressing through double-free detection, neighbor validation, and optional per-chunk canaries.

## Utility APIs (migration-4.3)

Include `<zephyr/sys/util_utf8.h>` for `utf8_trunc()` and `utf8_lcpy()` instead of relying on `util.h`. Rename `Z_MIN`, `Z_MAX`, and `Z_CLAMP` to `min`, `max`, and `clamp`.

## Watchdog startup (migration-4.4)

`CONFIG_WDT_DISABLE_AT_BOOT=n` no longer means a watchdog is automatically configured and running. Applications must configure it explicitly; the STM32, Raspberry Pi Pico, and TI `*_INITIAL_TIMEOUT` options used for the old behavior are removed.

