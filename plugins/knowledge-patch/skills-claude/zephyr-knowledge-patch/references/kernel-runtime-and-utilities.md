# Kernel, Runtime, and Utilities

Kernel and POSIX contracts, architecture support, power management, logging, utility APIs, and general-purpose subsystems.

## Architecture and execution protection

### Architecture current-pointer hooks (4.1.0)

Architecture ports can provide a custom current-thread implementation with `CONFIG_ARCH_HAS_CUSTOM_CURRENT_IMPL`. RISC-V can keep the current-thread pointer in the global pointer register with `CONFIG_RISCV_CURRENT_VIA_GP`.

### Architecture Kconfig changes (migration-4.2)

`CONFIG_SRAM_VECTOR_TABLE` now additionally depends on `CONFIG_XIP`, `CONFIG_ARCH_HAS_VECTOR_TABLE_RELOCATION`, and `CONFIG_ROMSTART_RELOCATION_ROM`. Rename the x86-only `CONFIG_DEBUG_INFO` option to `CONFIG_X86_DEBUG_INFO`.

### Architecture support and execution protection (4.2.0)

Zephyr gains initial Renesas RX support, including `rsk_rx130` and a QEMU-based target, while NIOS2 support is removed. With `CONFIG_ARM_MPU_PXN` and `CONFIG_USERSPACE`, `__ramfunc` and `__ram_text_reloc` are privileged-execute-never, so privileged code can no longer execute from those regions.

### Cache coherence API (migration-4.4)

Rename `CONFIG_ARCH_HAS_COHERENCE` to `CONFIG_CACHE_CAN_SAY_MEM_COHERENCE` and replace `arch_mem_coherent()` with `sys_cache_is_mem_coherent()`. Rename `CONFIG_CACHE_DOUBLEMAP` to `CONFIG_CACHE_HAS_MIRRORED_MEMORY_REGIONS`.

### Core API removals (4.0.0)

Replace `K_THREAD_STACK_MEMBER` with `K_KERNEL_STACK_MEMBER`, `ceiling_fraction` with `DIV_ROUND_UP`, the architecture CMSIS headers with `cmsis_core.h`, and `<zephyr/random/rand32.h>` with `<zephyr/random/random.h>`. `CBPRINTF_PACKAGE_COPY_*`, generated `_ENUM_TOKEN`/`_ENUM_UPPER_TOKEN`, deprecated `net_pkt` functions, and the `EARLY`, `APPLICATION`, and `SMP` device-init levels are gone; `net_buf_put()`/`net_buf_get()` and the kscan subsystem are deprecated.

### Hardware shadow stacks and Intel CET (4.3.0)

Zephyr adds architecture and kernel hardware-shadow-stack support through `CONFIG_ARCH_HAS_HW_SHADOW_STACK`, `CONFIG_HW_SHADOW_STACK`, sizing/declaration macros, and `k_thread_hw_shadow_stack_attach()`. x86 Intel CET and indirect-branch tracking are selected through the `CONFIG_X86_CET*` options.

### Loadable extensions and demand paging (4.0.0)

Devicetree devices are exported to LLEXT, and ARM64 gains initial LLEXT and demand-paging support. Demand paging also gains LRU eviction, SMP compatibility, and on-demand mappings through `CONFIG_DEMAND_MAPPING`.

### RISC-V fatal exception frames (4.0.0)

With `CONFIG_EXTRA_EXCEPTION_INFO`, `arch_esf` now has a `csf` pointer to the callee-saved registers for use by `k_sys_fatal_error_handler()`. SoCs selecting `RISCV_SOC_HAS_ISR_STACKING` must include that member in `SOC_ISR_STACKING_ESF_DECLARE`.

## General runtime and subsystem APIs

### Additional symbol migrations (4.4.0)

Replace `I2S_OPT_BIT_CLK_MASTER`/`I2S_OPT_FRAME_CLK_MASTER` with `I2S_OPT_BIT_CLK_CONTROLLER`/`I2S_OPT_FRAME_CLK_CONTROLLER`, and `I2S_OPT_BIT_CLK_SLAVE`/`I2S_OPT_FRAME_CLK_SLAVE` with `I2S_OPT_BIT_CLK_TARGET`/`I2S_OPT_FRAME_CLK_TARGET`; also replace `CONFIG_XOPEN_STREAMS` with `CONFIG_XSI_STREAMS` and `CONFIG_CTR_DRBG_CSPRNG_GENERATOR` with `CONFIG_PSA_CSPRNG_GENERATOR`. Correct `BT_HCI_LE_SUPERVISON_TIMEOUT_MIN`/`BT_HCI_LE_SUPERVISON_TIMEOUT_MAX` to `BT_HCI_LE_SUPERVISION_TIMEOUT_MIN`/`BT_HCI_LE_SUPERVISION_TIMEOUT_MAX`.

### CTF event identifiers (migration-4.4)

CTF metadata event IDs widen from 8 to 16 bits, permitting 65,535 events but making new traces incompatible with consumers expecting the old 8-bit format.

### Device initialization errors (migration-4.3)

`device_init()` now returns a negative `-errno` on initialization failure. Remove workarounds that interpreted the earlier erroneous positive value.

### Maximum CPU count (migration-4.0)

`CONFIG_MP_NUM_CPUS` was removed. Use `CONFIG_MP_MAX_NUM_CPUS`.

### MDIO lifecycle (migration-4.4)

`mdio_bus_enable()` and `mdio_bus_disable()` are removed because MDIO drivers now manage bus state internally.

### NXP EDMA discriminator (migration-4.4)

`CONFIG_DMA_MCUX_EDMA_V5` is removed now that EDMA v4 and v5 share one driver path; out-of-tree conditionals should use the unified `DMA_MCUX_EDMA_V4` handling.

### Other removed core interfaces (4.1.0)

Replace `CONFIG_PM_DEVICE_RUNTIME_EXCLUSIVE` with `CONFIG_PM_DEVICE_SYSTEM_MANAGED` and `z_arch_esf_t` with `struct arch_esf`. `z_pm_save_idle_exit()`, `CONFIG_WIFI_NM_WPA_SUPPLICANT_CRYPTO`, `CONFIG_NET_PKT_BUF_DATA_POOL_SIZE`, and `CONFIG_NET_TCP_ACK_TIMEOUT` are removed.

### Removed legacy options and moved SBC header (migration-4.4)

`CONFIG_JWT_SIGN_RSA_LEGACY` and `CONFIG_HAWKBIT_DDI_NO_SECURITY` are removed. Libsbc moves under Bluetooth, so include its header from `zephyr/bluetooth/sbc.h`.

### RTIO callback chains (migration-4.3)

RTIO callback operations gain an argument containing the first error result in the chain. Callbacks now run even when an earlier submission failed, so handlers must inspect that result instead of assuming prior success.

### Silabs RAIL and power-state configuration (migration-4.3)

Rename `CONFIG_RAIL_PA_CURVE_HEADER`, `CONFIG_RAIL_PA_CURVE_TYPES_HEADER`, and `CONFIG_RAIL_PA_ENABLE_CALIBRATION` to their `CONFIG_SILABS_SISDK_RAIL_*` forms. Series 2 SoCs remove the separate `em3` state and now choose EM2 or EM3 automatically from peripheral oscillator requests.

### Silabs Series 2 pin control (migration-4.1)

Series 2 devices use the new `silabs,dbus-pinctrl` driver, with signal macros from a SoC binding header and GPIO electrical properties expressed in devicetree groups:

```devicetree
group0 {
    pins = <I2C0_SDA_PD2>, <I2C0_SCL_PD3>;
    drive-open-drain;
    bias-pull-up;
};
```

### Streaming COBS and disjoint sets (4.4.0)

Incremental COBS processing uses `cobs_encoder_init()`, `cobs_encoder_write()`, and `cobs_encoder_close()`, with the matching `cobs_decoder_init()`/`cobs_decoder_write()`/`cobs_decoder_close()` lifecycle. The new `sys_set_node`, `sys_set_makeset()`, `sys_set_find()`, and `sys_set_union()` APIs provide disjoint-set operations.

### Suspend-to-RAM ownership (migration-4.3)

Applications must stop selecting `CONFIG_PM_S2RAM` and `PM_S2RAM_CUSTOM_MARKING`; SoCs and enabled `suspend-to-ram` devicetree power states now control them. Updated RW61x `exit-latency-us` values may also require increasing `min-residency-us` and can change power-state selection.

### zcbor 0.9 generated code (migration-4.0)

The generic `zcbor_simple_*()` APIs are removed; use `zcbor_bool_*()`, `zcbor_nil_*()`, or `zcbor_undefined_*()`. Regeneration may also capitalize additional C-keyword field names and rename bstr elements that use a `.size` specifier.

## Kernel scheduling, memory, and power

### Asynchronous runtime power management (4.2.0)

Device runtime PM can execute synchronously or asynchronously and can use the system workqueue or a dedicated workqueue. Configure this with `CONFIG_PM_DEVICE_RUNTIME_ASYNC`, `CONFIG_PM_DEVICE_RUNTIME_USE_SYSTEM_WQ`, or `CONFIG_PM_DEVICE_RUNTIME_USE_DEDICATED_WQ` and the dedicated-workqueue size, priority, and init-priority options.

### CPU load and frequency scaling (4.3.0)

The new `cpu_load` subsystem derives CPU-usage metrics from scheduler statistics. Experimental policy-driven dynamic clock scaling is selected with `CONFIG_CPU_FREQ` and can use those metrics to balance performance and power.

### Kernel shell commands (migration-4.0)

The `kernel threads` and `kernel stacks` commands are now `kernel thread list` and `kernel thread stacks`.

### Pressure-based CPU frequency policy (4.4.0)

The CPU frequency subsystem can select `CONFIG_CPU_FREQ_POLICY_PRESSURE` to scale frequency from scheduler load pressure.

### Reworked pipe API (migration-4.1)

The `CONFIG_PIPES` API is deprecated; the replacement pipe API is enabled automatically with `CONFIG_MULTITHREADING`. Replace `k_pipe_put()`/`k_pipe_get()` with `k_pipe_write()`/`k_pipe_read()`: `min_xfer` is gone, the byte count is returned directly, and threshold-based partial transfers are no longer supported.

Replace both flush calls with nonblocking `k_pipe_reset()`. Dynamic allocation through `k_pipe_alloc_init()`/`k_pipe_cleanup()` and availability queries are removed; `k_pipe_close()` instead closes a pipe and wakes waiters with an error, while buffered data remains readable until empty and `k_pipe_init()` reopens it.

### Runtime power-management defaults (4.3.0)

`CONFIG_PM_DEVICE_RUNTIME_DEFAULT_ENABLE` can enable device runtime power management by default, and drivers gain `pm_device_driver_deinit()` for deinitialization.

### Stable and heapless Zbus observers (4.2.0)

Zbus reaches stable API version 1.0.0. Runtime observer nodes can use dynamic, static, or no allocation through `CONFIG_ZBUS_RUNTIME_OBSERVERS_NODE_ALLOC_*`; the no-allocation mode registers caller-provided nodes with `zbus_chan_add_obs_with_node()`.

### Stack-canary strength (migration-4.1)

`CONFIG_STACK_CANARIES` no longer adds `-fstack-protector-all`. Enable `CONFIG_STACK_CANARIES_ALL` when all-function stack protection is required.

### Tiered heap hardening (4.4.0)

`CONFIG_SYS_HEAP_HARDENING` adds Basic, Moderate, Full, and Extreme checking for `sys_heap_alloc()` and `sys_heap_free()`, progressing through double-free detection, neighbor validation, and optional per-chunk canaries.

### Zbus asynchronous listeners and proxy agents (4.4.0)

`CONFIG_ZBUS_ASYNC_LISTENER` and `ZBUS_ASYNC_LISTENER_DEFINE()` run observers in a workqueue rather than the publisher thread, with queue selection through `zbus_async_listener_set_work_queue()`. Experimental `CONFIG_ZBUS_PROXY_AGENT` and `CONFIG_ZBUS_PROXY_AGENT_IPC`, with `ZBUS_PROXY_AGENT_DEFINE`, `ZBUS_PROXY_ADD_CHAN`, and shadow-channel macros, forward channels across CPU or domain boundaries over IPC.

## POSIX and file-descriptor contracts

### File-descriptor table sizing (migration-4.3)

`ZVFS_OPEN_SIZE` now determines file-descriptor table size and availability, with subsystem requirements contributed by `CONFIG_ZVFS_OPEN_ADD_SIZE_*`. `CONFIG_ZVFS_OPEN_MAX` remains but is raised to larger contributed minima unless `CONFIG_ZVFS_OPEN_IGNORE_MIN` is enabled.

### POSIX and kernel shell behavior (4.0.0)

The POSIX surface adds device I/O, signals, synchronized I/O, priority protection, `O_TRUNC`, `rmdir()`, `remove()`, and the reentrant time functions. The kernel shell can change thread CPU affinity at runtime, and bare `kernel reboot` now performs a cold reboot.

### POSIX and RISC-V Kconfig deprecations (4.3.0)

Rename `CONFIG_POSIX_READER_WRITER_LOCKS` to `CONFIG_POSIX_RW_LOCKS` and RISC-V's `CONFIG_EXTRA_EXCEPTION_INFO` to `CONFIG_EXCEPTION_DEBUG`; Newlib can opt into POSIX limits with `CONFIG_NEWLIB_LIBC_USE_POSIX_LIMITS_H`.

### POSIX headers and limits (migration-4.3)

Applications must include `<time.h>`, `<signal.h>`, and `<limits.h>` rather than the former `<zephyr/posix/...>` headers; non-POSIX C library ports may use Zephyr's `posix_time.h`, `posix_signal.h`, and `posix_limits.h`. Runtime-dependent limits may need to be obtained with `sysconf()`.

## Utilities, logging, state, and instrumentation

### Compiler-assisted instrumentation (4.3.0)

`CONFIG_INSTRUMENTATION` adds runtime call-graph tracing and statistical profiling through compiler-managed function instrumentation. It provides call-graph and statistical mode settings, trigger/stop and exclusion controls, and `instr_*` APIs for control and UART dumps.

### Rate-limited logging (4.3.0)

The `LOG_*_RATELIMIT` and `LOG_HEXDUMP_*_RATELIMIT` families rate-limit independently at each call site, using either `CONFIG_LOG_RATELIMIT_INTERVAL_MS` or an explicit rate. `CONFIG_LOG_RATELIMIT` controls the feature and `CONFIG_LOG_RATELIMIT_FALLBACK` selects log-all or drop-all behavior when it is disabled.

### Scope-based cleanup (4.4.0)

`SCOPE_VAR_DEFINE`, `SCOPE_GUARD_DEFINE`, and `SCOPE_DEFER_DEFINE`, with the `scope_var`, `scope_guard`, and `scope_defer` helpers, provide RAII/defer-style cleanup when C scope exits.

### State Machine Framework event propagation (migration-4.2)

`smf_set_handled()` is removed, and hierarchical state run actions now return `smf_state_result`: return `SMF_EVENT_HANDLED` to stop propagation or `SMF_EVENT_PROPAGATE` to invoke parent run actions. Flat state machines ignore the value, for which `SMF_EVENT_HANDLED` is the appropriate return.

### Utility APIs (migration-4.3)

Include `<zephyr/sys/util_utf8.h>` for `utf8_trunc()` and `utf8_lcpy()` instead of relying on `util.h`. Rename `Z_MIN`, `Z_MAX`, and `Z_CLAMP` to `min`, `max`, and `clamp`.
