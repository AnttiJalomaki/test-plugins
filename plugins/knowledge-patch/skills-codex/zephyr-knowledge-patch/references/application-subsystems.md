# Application Subsystems and Services

Use this reference for Zephyr work in this topic area. Entries are organized by developer task rather than release order.

## Additional symbol migrations (4.4.0)

Replace `I2S_OPT_BIT_CLK_MASTER`/`I2S_OPT_FRAME_CLK_MASTER` with `I2S_OPT_BIT_CLK_CONTROLLER`/`I2S_OPT_FRAME_CLK_CONTROLLER`, and `I2S_OPT_BIT_CLK_SLAVE`/`I2S_OPT_FRAME_CLK_SLAVE` with `I2S_OPT_BIT_CLK_TARGET`/`I2S_OPT_FRAME_CLK_TARGET`; also replace `CONFIG_XOPEN_STREAMS` with `CONFIG_XSI_STREAMS` and `CONFIG_CTR_DRBG_CSPRNG_GENERATOR` with `CONFIG_PSA_CSPRNG_GENERATOR`. Correct `BT_HCI_LE_SUPERVISON_TIMEOUT_MIN`/`BT_HCI_LE_SUPERVISON_TIMEOUT_MAX` to `BT_HCI_LE_SUPERVISION_TIMEOUT_MIN`/`BT_HCI_LE_SUPERVISION_TIMEOUT_MAX`.

## CTF event identifiers (migration-4.4)

CTF metadata event IDs widen from 8 to 16 bits, permitting 65,535 events but making new traces incompatible with consumers expecting the old 8-bit format.

## Modbus serial settings (migration-4.2)

Rename `modbus_serial_param.client_stop_bits` to `stop_bits`. Nonstandard stop-bit settings are disabled unless `CONFIG_MODBUS_NONCOMPLIANT_SERIAL_MODE` is enabled.

## Operational-amplifier subsystem (4.3.0)

`CONFIG_OPAMP` introduces a standard op-amp device API with initial Devicetree configuration and vendor-specific runtime configuration; initial compatibles are `nxp,opamp` and `nxp,opamp-fast`.

## Rate-limited logging (4.3.0)

The `LOG_*_RATELIMIT` and `LOG_HEXDUMP_*_RATELIMIT` families rate-limit independently at each call site, using either `CONFIG_LOG_RATELIMIT_INTERVAL_MS` or an explicit rate. `CONFIG_LOG_RATELIMIT` controls the feature and `CONFIG_LOG_RATELIMIT_FALLBACK` selects log-all or drop-all behavior when it is disabled.

## Stable and heapless Zbus observers (4.2.0)

Zbus reaches stable API version 1.0.0. Runtime observer nodes can use dynamic, static, or no allocation through `CONFIG_ZBUS_RUNTIME_OBSERVERS_NODE_ALLOC_*`; the no-allocation mode registers caller-provided nodes with `zbus_chan_add_obs_with_node()`.

## State Machine Framework event propagation (migration-4.2)

`smf_set_handled()` is removed, and hierarchical state run actions now return `smf_state_result`: return `SMF_EVENT_HANDLED` to stop propagation or `SMF_EVENT_PROPAGATE` to invoke parent run actions. Flat state machines ignore the value, for which `SMF_EVENT_HANDLED` is the appropriate return.

## Streaming COBS and disjoint sets (4.4.0)

Incremental COBS processing uses `cobs_encoder_init()`, `cobs_encoder_write()`, and `cobs_encoder_close()`, with the matching `cobs_decoder_init()`/`cobs_decoder_write()`/`cobs_decoder_close()` lifecycle. The new `sys_set_node`, `sys_set_makeset()`, `sys_set_find()`, and `sys_set_union()` APIs provide disjoint-set operations.

## Zbus asynchronous listeners and proxy agents (4.4.0)

`CONFIG_ZBUS_ASYNC_LISTENER` and `ZBUS_ASYNC_LISTENER_DEFINE()` run observers in a workqueue rather than the publisher thread, with queue selection through `zbus_async_listener_set_work_queue()`. Experimental `CONFIG_ZBUS_PROXY_AGENT` and `CONFIG_ZBUS_PROXY_AGENT_IPC`, with `ZBUS_PROXY_AGENT_DEFINE`, `ZBUS_PROXY_ADD_CHAN`, and shadow-channel macros, forward channels across CPU or domain boundaries over IPC.

## zcbor 0.9 generated code (migration-4.0)

The generic `zcbor_simple_*()` APIs are removed; use `zcbor_bool_*()`, `zcbor_nil_*()`, or `zcbor_undefined_*()`. Regeneration may also capitalize additional C-keyword field names and rename bstr elements that use a `.size` specifier.

