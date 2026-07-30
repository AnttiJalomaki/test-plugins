---
name: zephyr-knowledge-patch
description: Zephyr RTOS
version: 4.4.0
license: MIT
metadata:
  author: Nevaberry
---


# Zephyr RTOS Compatibility Guide

Use this skill when writing, migrating, reviewing, or debugging Zephyr applications,
boards, drivers, subsystems, Devicetree, Kconfig, west/sysbuild flows, networking,
Bluetooth, security, storage, or tests.

## How to use this skill

1. Inspect the application's manifest, board target, Kconfig fragments, Devicetree
   overlays, and west/sysbuild invocation before proposing a migration.
2. Read the quick-reference hazards below first. They identify changes that can
   compile successfully while changing boot, flash, security, storage, or protocol
   behavior.
3. Open only the topic references relevant to the task. Each reference is organized
   by developer activity, with the originating batch ID attached to every entry.
4. Prefer the project's source, generated configuration, devicetree, and tests when
   they demonstrate behavior more specific than this general guidance.
5. Preserve explicit configuration during migrations. Many former implicit selects,
   defaults, and transitive includes now require direct declarations.

## Reference index

| Reference | Topics |
| --- | --- |
| [application-subsystems.md](references/application-subsystems.md) | Zbus, SMF, COBS, zcbor, logging, tracing, and miscellaneous subsystem APIs |
| [bluetooth.md](references/bluetooth.md) | Host/controller contracts, LE Audio, Mesh, pairing, HCI, ISO, and GATT |
| [build-boards-testing.md](references/build-boards-testing.md) | West, sysbuild, runners, board targets, HWMv2, toolchains, Twister, and ztest |
| [devicetree-drivers.md](references/devicetree-drivers.md) | Binding migrations, driver APIs, clocks, buses, storage hardware, SoCs, and peripherals |
| [graphics-input-usb.md](references/graphics-input-usb.md) | Display, video, LVGL, input, stepper, haptics, PWM, USB device, and USB host |
| [kernel-architecture.md](references/kernel-architecture.md) | Kernel APIs, POSIX, scheduling, memory, runtime PM, architecture, and timing |
| [networking-protocols.md](references/networking-protocols.md) | Sockets, Ethernet, Wi-Fi, OpenThread, HTTP, CoAP, MQTT, LwM2M, and modem protocols |
| [security-storage-update.md](references/security-storage-update.md) | MCUboot, TF-M, PSA Crypto, Mbed TLS, secure storage, flash, filesystems, and MCUmgr |

## Start with these breaking changes

### Board and build floor

- HWMv1 is gone. Convert out-of-tree boards and SoCs to HWMv2; compatibility board
  aliases are not available as a fallback.
- Zephyr 4.4 requires Zephyr SDK 1.0.0 and Python 3.12 and defaults to C17.
  `CONFIG_STD_C99` and `CONFIG_STD_C11` are temporary deprecated fallbacks for
  toolchains without C17.
- A new `board.yml` entry needs `full_name`. `BOARD_QUALIFIERS` has no leading `/`,
  so form qualified targets with `${BOARD}/${BOARD_QUALIFIERS}`.
- `SNIPPET_ROOT` no longer includes the application directory automatically. Declare
  `snippet_root` in `zephyr/module.yml` or append it from CMake.
- Downstream libraries that include generated heap constants must depend on
  `zephyr_generated_headers` to avoid a parallel-build race.

### Flashing and boot images

- The build no longer invokes `west sign` for MCUboot. Put imgtool arguments in
  `CONFIG_MCUBOOT_EXTRA_IMGTOOL_ARGS`.
- MCUboot defaults to swap-using-offset. With unequal slots, make the secondary slot
  one sector larger than the primary, or explicitly retain
  `SB_CONFIG_MCUBOOT_MODE_SWAP_USING_MOVE`.
- BOSSA does not erase the whole device unless `west flash --erase` is passed.
  Nordic runner defaults and erase behavior also changed; make destructive flashing
  intent explicit in automation.
- TF-M BL2 FOTA BIN images are unconfirmed. Accept them with `psa_fwu_accept()` or
  they can roll back after reboot.
- Several board flash layouts changed incompatibly. Do not assume an image can be
  upgraded in place merely because it builds for the same board name.

### Security and persistent data

- Mbed TLS now supplies TLS/X.509 while TF-PSA-Crypto supplies cryptography. Legacy
  Mbed TLS crypto configuration, DES, selected curves, and algorithm-level toggles
  are not compatibility paths.
- PSA Crypto is the supported path for Mesh, OpenThread, JWT, UpdateHub, MCUmgr
  hashing, and flash-integrity work. Select providers and storage backends explicitly.
- `psa_storage_uid_t` changed from 64 to 30 bits. Use
  `CONFIG_SECURE_STORAGE_64_BIT_UID` when installed devices must authenticate data
  written with the older UID width.
- Hardware rollback protection takes `CONFIG_TFM_IMAGE_SECURITY_COUNTER` directly.
  Set it deliberately rather than deriving it from the image version.
- Secure-storage backends no longer pull in Settings or NVS implicitly. Enable
  Settings, NVS, ZMS, or a custom backend according to the deployment.

### Devicetree and driver contracts

- Binding property names must be hyphenated. Binding defaults for `status`,
  `#address-cells`, and `#size-cells` are build errors; place values in DTS.
- `DT_REG_ADDR*` and `DT_REG_SIZE*` produce unsigned literals. Use
  `DT_REG_ADDR_RAW` when an address is used as a Devicetree index.
- Out-of-tree implementations of upstream driver classes must declare APIs with
  `DEVICE_API()` so iterable-section validation can recognize them.
- Device initialization failures return negative `-errno`. Remove workarounds for
  the former positive failure value.
- Driver settings increasingly belong in Devicetree: clocks, PHY linkage, flash
  topology, card detection, power supplies, regulators, and chip-select delays all
  have migrations detailed in the hardware reference.

### Kernel and POSIX

- Replace the deprecated pipe calls with `k_pipe_write()`, `k_pipe_read()`,
  `k_pipe_reset()`, and `k_pipe_close()`. Threshold transfers and dynamic pipe
  allocation are gone.
- Include standard POSIX headers such as `<time.h>`, `<signal.h>`, `<limits.h>`, and
  `<sys/socket.h>` directly. Zephyr wrapper and networking headers no longer provide
  all names transitively.
- `CONFIG_STACK_CANARIES` no longer implies protection for every function; select
  `CONFIG_STACK_CANARIES_ALL` when that policy is required.
- Runtime PM may be synchronous or asynchronous and can use the system or a dedicated
  workqueue. Audit callback context and ordering before enabling it globally.
- A watchdog is not automatically configured merely because
  `CONFIG_WDT_DISABLE_AT_BOOT=n`; applications must configure it explicitly.

### Bluetooth

- LE Audio services increasingly require runtime registration. Register PACS before
  ASCS, register Unicast Server before callbacks, and explicitly enable GATT, SMP,
  advertising, synchronization, ISO, and PAC features the application uses.
- Extended advertising never auto-resumes. Restart it explicitly after connections;
  legacy applications that depended on automatic resumption need the same audit.
- Legacy LE passkey entry no longer grants MITM authentication. Secure the ACL with
  `bt_conn_set_security()` before connecting ISO channels.
- HCI buffers now carry the H:4 type in their payload, commands are allocated with
  `bt_hci_cmd_alloc()`, and ISO data paths must be set up and removed explicitly.
- Bluetooth Mesh uses PSA Crypto. Changing providers can make stored keys
  incompatible, requiring unprovisioning and reprovisioning.

### Networking

- Zephyr network names now use `net_`, `NET_`, or `ZSOCK_` namespaces. Treat
  `net_compat.h` as a migration aid, not a permanent include strategy.
- Raw packet sockets use `AF_PACKET/SOCK_DGRAM/ETH_P_ALL`; the former
  `AF_PACKET/SOCK_RAW/IPPROTO_RAW` tuple is invalid.
- Dynamic HTTP callbacks have revised request contexts and transaction lifecycle
  values. Reset per-request state on `HTTP_SERVER_TRANSACTION_COMPLETE`.
- Secure-socket protocol is enforced as the minimum TLS version, and socket TLS
  support no longer selects ciphers or protocol configuration automatically.
- Network-management event values are 64-bit. Decode them with
  `NET_MGMT_LAYER_CODE` and `NET_MGMT_GET_COMMAND`.

### Video, display, input, and USB

- USB device_next is the default device stack. Out-of-tree UDC drivers must let the
  stack own control-transfer buffers, and UVC uses an explicit initialize, add-format,
  enable, and shutdown lifecycle.
- Video stream hooks require `video_buf_type`; drivers set pitch, applications size
  buffers from `video_format.size`, and allocations take a timeout.
- Several pixel-format names and byte-layout meanings changed. Remove RGB/BGR and
  monochrome inversion workarounds only after verifying the actual buffer layout.
- Stepper APIs evolve first to `stepper_move_by()`/`stepper_move_to()` and then to the
  `stepper_ctrl_*` family. Controller nodes now own step/direction motion properties.
- Input callbacks and shell bypass callbacks gain user-data pointers; pass `NULL`
  when no context is needed.

## High-value current facilities

### Storage, isolation, and update

- Secure Storage exposes PSA storage and persistent PSA keys broadly; ZMS supports
  conventional NOR plus no-erase media, and NVMEM supports EEPROM and flash backends.
- OTP provides standard read/program operations, with programming separately gated.
- Tiered heap hardening, hardware shadow stacks, Intel CET, privileged-execute-never
  RAM regions, and in-memory core dumps offer progressively stronger diagnostics and
  execution protection.
- MCUmgr supports discovery, slot information, LoRaWAN, UDP/DTLS, and raw UART
  transports, plus firmware-updater and retention-boot flows.

### Connectivity

- HTTP supports dynamic methods, diagnostics, fallback resources, compression, and
  request-scoped contexts suitable for concurrent HTTP/2 streams.
- MQTT 5, MQTT-SN discovery, secure CoAP services, FTP client, WireGuard, Wi-Fi
  Direct, MCTP over I3C, and parallel-client DTLS server operation are available.
- OpenThread can run as a standalone module and provides NAT64, border-router,
  wake-up, prefix-delegation, multi-instance, and radio-coexistence facilities.
- LwM2M adds scheduled sends, richer IPSO objects, numerical observation controls,
  bootstrap fallback, and cache inspection/filtering.

### Core and application infrastructure

- CPU load and CPU-frequency policy APIs support metric-driven and pressure-driven
  scaling. Compiler instrumentation provides call-graph and statistical profiling.
- Zbus provides stable heapless observer registration, asynchronous listeners, and
  experimental IPC proxy agents.
- Scope guards provide RAII/defer-style cleanup in C, while streaming COBS and
  disjoint-set utilities cover incremental framing and union/find workloads.
- The build dashboard, `dtdoctor`, `traceconfig`, footprint charts, and ztest
  benchmarks improve configuration and performance diagnosis.

### Hardware and media

- Standard device classes include comparator, haptics, stepper, op-amp, biometrics,
  wake-up controller, OTP, and richer NVMEM access.
- Video supports multi-heap and named-region buffers, imported storage, format
  classifiers, multiple displays, V4L2-style controls, and USB host/device UVC.
- Devicetree register iteration, integer unit-address lookup, and first-class nexus
  mappings reduce custom generated-code machinery.
- Selective Ethernet statistics avoid expensive vendor firmware queries, while PWM,
  display, and haptics event callbacks support asynchronous notification.

## Migration review checklist

- Confirm the exact board target and qualifiers after HWMv2 conversion.
- Run `west update` when module requirements changed, especially CMSIS 6 and TF-M.
- Regenerate Devicetree bindings and inspect warnings for renamed properties,
  compatibles, enum representation, clocks, and chosen nodes.
- Compare old and new `.config` files for symbols that stopped selecting dependencies.
- Inspect generated partition and runner configuration before flashing real hardware.
- Test MCUboot confirmation, rollback, encrypted images, and retained secure storage on
  a disposable device before deploying an upgrade.
- Exercise disconnect/reconnect, advertising restart, pairing, and LE Audio runtime
  registration paths under Bluetooth tests.
- Exercise HTTP transaction completion, socket TLS settings, DNS source selection,
  network-management decoding, and raw-socket creation under integration tests.
- Recompile all callers when public structure layout changed; do not rely on ABI
  compatibility for Wi-Fi, video, cellular, Bluetooth, or network event structures.
- Run Twister with current list syntax and identifiers, then validate memory footprint,
  stack protection, watchdog startup, and runtime-PM behavior on target hardware.
