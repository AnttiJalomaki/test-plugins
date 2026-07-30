---
name: zephyr-knowledge-patch
description: Zephyr RTOS
version: 4.4.0
license: MIT
metadata:
  author: Nevaberry
---


# Zephyr RTOS Knowledge Patch

Use this skill when upgrading, building, porting, or debugging a Zephyr
application, board, driver, module, network service, Bluetooth product, or
firmware-update flow. Start with the migration triage below, then open the
reference for the affected subsystem. Apply related Kconfig, Devicetree,
callback, structure, and runtime changes together; many compile-time renames
also carry behavioral changes.

## Reference index

| Reference | Topics |
| --- | --- |
| [build-boards-and-testing.md](references/build-boards-and-testing.md) | Build configuration, toolchains, modules, board targets, runners, native simulation, Twister, ztest, and diagnostics |
| [devicetree-and-hardware.md](references/devicetree-and-hardware.md) | Binding syntax, properties, generated macros, clocks, regulators, partitions, and SoC hardware description |
| [drivers-and-peripherals.md](references/drivers-and-peripherals.md) | ADC, CAN, I3C, SPI, UART, GPIO, DMA, flash, storage devices, timers, counters, watchdogs, and driver APIs |
| [kernel-runtime-and-utilities.md](references/kernel-runtime-and-utilities.md) | Kernel, POSIX, architecture, power management, Zbus, logging, state machines, instrumentation, and utilities |
| [networking-and-iot.md](references/networking-and-iot.md) | Ethernet, sockets, HTTP, CoAP, MQTT, LwM2M, OpenThread, Wi-Fi, LoRa, Modbus, and network management |
| [bluetooth.md](references/bluetooth.md) | Host, controller, HCI, connections, pairing, Mesh, Classic, LE Audio, and profile registration |
| [security-storage-and-updates.md](references/security-storage-and-updates.md) | Mbed TLS, TF-PSA-Crypto, secure storage, NVMEM, MCUboot, TF-M, MCUmgr, hawkBit, and persistence |
| [display-input-and-media.md](references/display-input-and-media.md) | Display, LVGL, video, UVC, input, haptics, USB media, and stepper control |

## Breaking-change triage

Check an upgrade in this order:

1. Confirm the host toolchain and language floor, then update modules with
   `west update`. Check Python, SDK, C language mode, CMSIS, and generated-header
   dependencies before diagnosing application code.
2. Convert all out-of-tree boards and SoCs to HWMv2 and use current qualified
   board targets. Recheck runner defaults and destructive flash options.
3. Run a pristine configure so removed Kconfig symbols, binding errors, and
   generated Devicetree changes are visible. Do not preserve old defaults by
   accident.
4. Audit callback signatures, structure layouts, enum values, and ownership.
   Several APIs add user data, widths, status values, family arguments, or
   lifecycle calls while retaining familiar names.
5. Revalidate security and update state on real devices. Crypto-provider,
   secure-storage UID, image-slot, flash-layout, and pairing changes can make
   previously persisted data or images incompatible.
6. Exercise runtime behavior on hardware. Advertising restart, watchdog
   startup, power management, PHY configuration, HTTP transaction completion,
   and video negotiation are not compile-only migrations.

## Build, board, and Devicetree essentials

### Establish the build environment

- Use Zephyr SDK 1.0.0 or newer and Python 3.12 or newer. C17 is the default;
  `CONFIG_STD_C99` and `CONFIG_STD_C11` are temporary deprecated fallbacks for
  toolchains that cannot compile C17.
- Add `zephyr_generated_headers` as a dependency of downstream CMake libraries
  that include `kernel.h` and might race generation of `heap_constants.h`.
- Add the application directory to `SNIPPET_ROOT` explicitly through
  `zephyr/module.yml` or CMake when it supplies snippets.
- New `board.yml` entries require `full_name`. `BOARD_QUALIFIERS` has no leading
  slash, so form a qualified name as `${BOARD}/${BOARD_QUALIFIERS}`.

### Treat hardware descriptions as authoritative

- HWMv1 is gone. Port out-of-tree hardware to HWMv2 and remove old alias board
  names rather than depending on compatibility targets.
- `DT_REG_ADDR*` and `DT_REG_SIZE*` produce unsigned literals. Use
  `DT_REG_ADDR_RAW` when an address must serve as a Devicetree index.
- Bindings cannot default `status`, `#address-cells`, or `#size-cells`; put
  required values in the Devicetree source.
- Local binding property names use hyphens, not underscores. Run the binding
  migration utility for broad mechanical conversions, then review units and
  semantics manually.
- Replace selected `zephyr,code-partition` nodes with
  `zephyr,mapped-partition`; the unit address is the mapped address and the
  node cannot be combined with `CONFIG_FLASH_LOAD_OFFSET` or
  `CONFIG_FLASH_LOAD_SIZE`.
- Out-of-tree implementations of upstream driver classes declare their API
  with `DEVICE_API()` so iterable-section validation and `DEVICE_API_IS` work.

## Kernel and core API essentials

### Replace removed interfaces deliberately

- Use `K_KERNEL_STACK_MEMBER`, `DIV_ROUND_UP`, `cmsis_core.h`, and
  `<zephyr/random/random.h>` in place of their removed predecessors. Audit
  generated enum-token helpers, init levels, `net_pkt`, and `net_buf` calls in
  the same pass.
- Replace the deprecated pipe API with `k_pipe_write()`, `k_pipe_read()`,
  `k_pipe_reset()`, and `k_pipe_close()`. Partial-transfer thresholds,
  availability queries, and dynamic pipe allocation are not preserved.
- Treat `device_init()` failure as negative `-errno`. Remove workarounds that
  expected a positive failure value.
- Include standard `<time.h>`, `<signal.h>`, and `<limits.h>` for POSIX code;
  use `sysconf()` for limits that vary at runtime.
- Enable `CONFIG_POLL` explicitly when Bluetooth-host or other application
  code calls `k_poll`; it is no longer selected transitively.
- Enable `CONFIG_STACK_CANARIES_ALL` when all-function protection is required;
  `CONFIG_STACK_CANARIES` alone no longer adds `-fstack-protector-all`.

### Recheck power and execution behavior

- Device runtime PM can be synchronous or asynchronous and can use the system
  or a dedicated workqueue. Select the workqueue and sizing options explicitly.
- SoCs and enabled `suspend-to-ram` Devicetree states own S2RAM selection;
  applications should not select the removed ownership symbols.
- Watchdogs are not automatically configured merely because
  `CONFIG_WDT_DISABLE_AT_BOOT=n`. Configure and start the watchdog in the
  application.
- Review execute-never regions, hardware shadow stacks, cache-coherence tests,
  exception-frame fields, and architecture current-thread hooks when porting
  architecture or SoC code.

## Security, storage, and firmware-update essentials

### Migrate crypto as a provider stack

- Treat Mbed TLS as the TLS/X.509 layer and TF-PSA-Crypto as the cryptographic
  provider. Legacy Mbed TLS crypto configuration, custom configuration files,
  and old algorithm-level toggles are not supported by the split stack.
- Select TLS versions and ciphersuites explicitly. Secure-socket creation now
  enforces the protocol argument as the minimum TLS version, and socket TLS no
  longer pulls crypto settings in transitively.
- `CONFIG_PSA_CRYPTO` chooses TF-M for TF-M builds and Mbed TLS otherwise;
  use the custom-provider choice only when the application supplies the full
  provider contract.
- Persistent PSA key identifiers come from the allocated user and subsystem
  ranges in `<zephyr/psa/key_ids.h>`. Size key slots explicitly where needed.
- Existing secure-storage records created with 64-bit UIDs need
  `CONFIG_SECURE_STORAGE_64_BIT_UID` during an upgrade that must preserve them.

### Revalidate boot and update flows

- The build no longer invokes `west sign` for MCUboot. Pass imgtool arguments
  through `CONFIG_MCUBOOT_EXTRA_IMGTOOL_ARGS`.
- MCUboot defaults to swap-using-offset. Recheck primary and secondary slot
  geometry, or explicitly retain swap-using-move.
- Do not derive the TF-M rollback counter from an image version. Set
  `CONFIG_TFM_IMAGE_SECURITY_COUNTER` deliberately.
- TF-M BL2 signing derives alignment and sector counts from the hardware
  description. Its secure and non-secure BIN outputs are unconfirmed FOTA
  images and need `psa_fwu_accept()` to prevent rollback after reboot.
- Backends no longer select secure-storage dependencies. Enable Settings and
  NVS, ZMS, TF-M PSA storage, or a custom backend explicitly.

## Networking essentials

### Update type and namespace contracts

- Include network buffers from `<zephyr/net_buf.h>`. Zephyr networking symbols
  use `net_`, `NET_`, and `ZSOCK_` prefixes; POSIX applications include POSIX
  socket headers directly instead of receiving names transitively.
- `net_mgmt` event and request callbacks receive a `uint64_t` event. Decode it
  with `NET_MGMT_LAYER_CODE` and `NET_MGMT_GET_COMMAND`.
- The inline `net_linkaddr` replaces pointer-based link-address storage. Test
  `len == 0`, not `addr == NULL`, for an unset address.
- Use packet datagram sockets for link-layer packets; the former
  `AF_PACKET/SOCK_RAW/IPPROTO_RAW` combination is invalid.

### Honor server and transaction lifecycles

- Size HTTP service concurrency and backlog values deliberately and provide
  the required configuration and fallback-resource arguments.
- Dynamic HTTP handlers use request contexts and transaction status values.
  Handle transaction completion to reset per-request state after the response
  has actually been sent.
- HTTP response callbacks return `0` to continue or nonzero to abort.
- CoAP client request paths and options are embedded arrays, and response
  callbacks receive a `coap_client_response_data` object. Move discovery
  attributes from `coap_resource.user_data` to `metadata`.
- MQTT 3.1.1 callers pass `NULL` to the extended `mqtt_disconnect()`; enable
  the MQTT 5 option before using MQTT 5 properties.

## Bluetooth essentials

### Make registration and dependencies explicit

- Register the Unicast Server before callbacks, PACS before ASCS, and the Scan
  Delegator through its runtime register function. Use the corresponding
  unregister functions for teardown.
- LE Audio no longer selects GATT client, CCC discovery/update, extended or
  periodic advertising, ISO broadcast/sync, PAC source/sink, or SMP features
  automatically. Enable exactly what the product uses.
- Multiple callback registrations are supported by several host and audio
  APIs; do not retain single-listener storage assumptions.
- Extended advertising never auto-resumes. Restart it explicitly after a
  connection, and use the current connectable option or shorthand.

### Recheck security and data paths

- Legacy LE passkey-entry pairing no longer provides MITM authentication, and
  loaded bonds created that way are downgraded. Request the required ACL
  security explicitly before connecting an ISO channel.
- Applications create and remove ISO data paths explicitly. Test central and
  peripheral ISO channel types instead of the removed generic connected type.
- Mesh PSA migration can make TinyCrypt-provisioned keys incompatible. An
  in-place provider switch may require unprovisioning and reprovisioning.
- Use application passkey callbacks instead of the deprecated fixed-passkey
  Kconfig path.

## Display, video, and motion essentials

- Supply `video_buf_type` to stream operations, use `video_format.size` for
  allocation, and let drivers set pitch. Bits-per-pixel helpers return bits,
  not bytes.
- UVC no longer configures the source device automatically after host
  negotiation. Apply the selected frame format and rate in the application.
- Use `uvc_device_init()`, `uvc_device_add_format()`, `uvc_device_enable()`,
  and `uvc_device_shutdown()` for the current UVC lifecycle.
- Motion control uses the `stepper_ctrl_*` API and controller event types.
  Put step/dir motion properties on a `zephyr,gpio-step-dir-stepper-ctrl` node.
- Recheck RGB/BGR and monochrome workarounds. Several pixel-format identifiers
  and byte-layout meanings were corrected rather than merely renamed.

## High-value capabilities

- Secure storage provides the PSA storage surface and persistent PSA keys;
  ZMS supports both erase-based NOR and no-erase nonvolatile media.
- NVMEM exposes named or indexed cells and supports EEPROM and flash-backed
  implementations; OTP adds standard read and gated program operations.
- WireGuard, Wi-Fi Direct, multicast CoAP clients, FTP clients, parallel-client
  DTLS servers, and an OpenThread NAT64 translator expand network integration.
- Asynchronous Zbus listeners move observer work out of publisher context, and
  proxy agents can forward channels over IPC between domains.
- CPU-frequency policies can react to scheduler load or pressure, while
  `cpu_load` exposes usage metrics from scheduler statistics.
- Scope guards and deferred cleanup provide structured C cleanup, and tiered
  heap hardening adds progressively stronger allocator validation.
- The USB host-class framework can drive UVC cameras, while the device stack
  supports runtime configuration and multiple simultaneous controllers.

## Migration workflow

1. Record the exact board target, runner, modules, toolchain, application
   configuration, overlays, and sysbuild inputs.
2. Perform a pristine configure and classify failures by reference topic.
3. Apply a whole subsystem migration: names, types, callbacks, dependencies,
   Devicetree, defaults, persistence, and lifecycle.
4. Inspect generated configuration and Devicetree output before flashing.
5. Exercise initialization failures, reconnects, suspend/resume, update and
   rollback, persisted state, protocol aborts, and teardown paths on hardware.
6. Run Twister or ztest coverage and any host-side parsers against the new
   identifiers and output formats.
