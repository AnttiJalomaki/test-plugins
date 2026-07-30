# Drivers and Peripherals

Driver API migrations and peripheral contracts for buses, storage, sensors, timers, GPIO, CAN, flash, and related hardware.

## Analog, sensing, and biometrics

### ADC migrations (migration-4.4)

Rename `renesas,ra-adc` to `renesas,ra-adc12`, `CONFIG_ADC_MCUX_SAR_ADC` to `CONFIG_ADC_NXP_SAR_ADC`, and STM32 ADC `resolutions` to `st,adc-resolutions`. NXP SAR ADC nodes must add `zephyr,input-positive`, and supported SoCs should use `ADC_REF_VDD_1` rather than `ADC_REF_INTERNAL`.

### ADC sequence priority (4.4.0)

`adc_sequence.priority`, enabled with `CONFIG_ADC_SEQUENCE_PRIORITY`, lets ADC requests carry an explicit sequence priority.

### Biometrics and wake-up controllers (4.4.0)

Zephyr adds standard biometrics and Wake-up Controller device classes. Initial biometrics bindings are `adh-tech,gt5x`, `zhiantec,zfm-x0`, and `zephyr,biometrics-emul`; the initial WUC binding is `nxp,llwu`.

### Operational-amplifier subsystem (4.3.0)

`CONFIG_OPAMP` introduces a standard op-amp device API with initial Devicetree configuration and vendor-specific runtime configuration; initial compatibles are `nxp,opamp` and `nxp,opamp-fast`.

## Buses, GPIO, DMA, and messaging

### CAN API removals (4.1.0)

Replace `CAN_MAX_STD_ID` and `CAN_MAX_EXT_ID` with `CAN_STD_ID_MASK` and `CAN_EXT_ID_MASK`, and replace `can_get_min_bitrate()` and `can_get_max_bitrate()` with `can_get_bitrate_min()` and `can_get_bitrate_max()`. `can_calc_prescaler()` is removed without a listed replacement.

### CAN capacity configuration (migration-4.4)

The generic `CONFIG_CAN_MAX_FILTER`, `CONFIG_CAN_MAX_STD_ID_FILTER`, and `CONFIG_CAN_MAX_EXT_ID_FILTER` options are removed; configure the driver-specific limit, such as `CONFIG_CAN_MCUX_FLEXCAN_MAX_FILTERS` or the STM32 BXCAN/FDCAN standard and extended filter limits. FlexCAN's `CONFIG_CAN_MAX_MB` likewise moves to the per-instance `number-of-mb` Devicetree property.

### CAN driver settings (migration-4.4)

Rename `CONFIG_CAN_MCUX_MCAN` to `CONFIG_CAN_NXP_LPC_MCAN`. A `ti,tcan4x5x` node must set `ti,nwkrq-voltage-vio` to retain the former VIO default, while an `nxp,flexcan` `clk-source` now selects between the named `clksrc0` and `clksrc1` inputs.

### Connector and Nordic UART behavior (4.2.0)

Boards with Qwiic, Stemma, or Grove I2C connectors now expose the common `zephyr_i2c` devicetree label, allowing connectorized I2C shields to work across branding through `west build --shield`. The Nordic UART receiver mode that uses an extra timer is no longer deprecated because it is the reliable receive path without hardware flow control.

### DMA userspace access (migration-4.3)

The DMA API no longer exposes userspace syscalls because their access and parameter verification could not be made safe. Userspace code can no longer invoke the DMA API through the former syscall surface.

### I3C address-management API (4.0.0)

`i3c_ccc_do_setdasa()` now takes the dynamic address explicitly, `i3c_determine_default_addr()` is removed, and `attach_i3c_device()` no longer takes an address because the driver derives it from `i3c_device_desc`. Controllers may advertise SETAASA support with the `supports-setaasa` devicetree property.

### I3C target, RTIO, and controller handoff (4.1.0)

New I3C surfaces include `CONFIG_I3C_TARGET_BUFFER_MODE`, `CONFIG_I3C_RTIO`, `i3c_ibi_hj_response()`, `i3c_ccc_do_getacccr()`, and `i3c_device_controller_handoff()`. Initial controller bindings include `snps,designware-i3c` and `st,stm32-i3c`.

### Native PTY UART (migration-4.2)

`uart_native_posix` becomes `uart_native_pty`, with `zephyr,native-pty-uart`, `CONFIG_UART_NATIVE_PTY*`, and `CONFIG_UART_NATIVE_PTY_0` replacing the old POSIX names. Instantiate one devicetree node per UART; at runtime, `--<uart_name>_stdinout` connects an instance to standard input/output instead of a PTY.

### Raspberry Pi Pico PWM division (migration-4.0)

The Pico PWM driver now chooses its frequency division adaptively when the channel divider is omitted or zero. Set a nonzero `divider-int-0` (or the corresponding channel property) explicitly when fixed, pre-4.0 division behavior is required.

### Scheduler and I3C configuration (4.2.0)

Replace deprecated `CONFIG_SCHED_DUMB` and `CONFIG_WAITQ_DUMB` with `CONFIG_SCHED_SIMPLE` and `CONFIG_WAITQ_SIMPLE`. I3C group addressing and `CONFIG_I3C_USE_GROUP_ADDR` are removed; choose `CONFIG_I3C_CONTROLLER_ROLE_ONLY`, `CONFIG_I3C_TARGET_ROLE_ONLY`, or `CONFIG_I3C_DUAL_ROLE` through `CONFIG_I3C_MODE`.

### Signal-only IPM callbacks (4.4.0)

Mailbox-backed IPM now supports signal-only messages; callbacks must accept a `NULL` payload when the mailbox supplies no data buffer.

### SPI chip-select timing (migration-4.3)

`SPI_CS_CONTROL_INIT*`, `SPI_CONFIG_DT*`, and `SPI_DT_SPEC_GET*` no longer accept a delay argument. Put peripheral chip-select timing in devicetree with `spi-cs-setup-delay-ns` and `spi-cs-hold-delay-ns`.

### SPI NOR deep power-down (migration-4.0)

`CONFIG_SPI_NOR_IDLE_IN_DPD` is removed. Enable device runtime power management for the SPI NOR device and tune the replacement behavior with `CONFIG_SPI_NOR_ACTIVE_DWELL_MS`.

### STM32 SPI and interrupt sizing (migration-4.4)

`CONFIG_SPI_STM32_USE_HW_SS` is removed: `cs-gpios` or `st,soft-nss` selects Soft NSS, and their absence selects Hard NSS. SPI pins now default to `very-high-speed` slew and may need an overlay to reduce power, while auto-derived `CONFIG_NUM_IRQS` may need an explicit larger value for IRQs used only through `IRQ_CONNECT()`.

### UART readiness and LiteX Kconfig (migration-4.0)

Treat `uart_irq_tx_ready()` as ready when its return value is greater than zero, not only when it equals one; the value is now a lower bound on bytes accepted by `uart_fifo_fill()`. Rename `CONFIG_UART_LITEUART` to `CONFIG_UART_LITEX`.

## Flash, memory, and storage devices

### EEPROM target access (migration-4.4)

I2C EEPROM target code should replace deprecated `eeprom_target_program()` with `eeprom_target_read_data()` and `eeprom_target_write_data()`, which take explicit offset and length arguments.

### NAND flash translation and bad blocks (4.4.0)

The `zephyr,ftl-dhara` disk driver exposes NAND as a standard disk with wear leveling and bad-block management. Flash drivers can implement `FLASH_EX_OP_MARK_BAD_BLOCK` and `FLASH_EX_OP_IS_BAD_BLOCK`, while `jedec,mspi-nor` can configure read, write, and control commands separately in Devicetree.

### NXP FlexSPI NOR topology (migration-4.4)

An `nxp,imx-flexspi-nor` node is now a controller containing a `soc-nv-flash` child; move erase/write sizes and partitions to the child and add `ranges` to the controller. Point `zephyr,flash` at the child and `zephyr,flash-controller` at the controller.

### NXP USDHC card detection (migration-4.0)

Without a configured card-detect method, NXP USDHC now assumes a card is present. Add `detect-cd` to the active USDHC node to keep using the peripheral's internal card-detect signal.

### PSRAM refresh configuration (migration-4.4)

`st,stm32-xspi-psram` and `st,stm32-ospi-psram` nodes must set `st,refresh` in memory-clock cycles; their hard-coded defaults are gone.

### Renesas RA flash naming (migration-4.2)

Rename `CONFIG_RA_FLASH_HP` to `CONFIG_SOC_FLASH_RENESAS_RA_HP` and `CONFIG_FLASH_RA_WRITE_PROTECT` to `CONFIG_FLASH_RENESAS_RA_HP_WRITE_PROTECT`; `CONFIG_DUAL_BANK_MODE` is removed. The generic `renesas,ra-nv-flash` binding is split into `renesas,ra-nv-code-flash` and `renesas,ra-nv-data-flash`.

### SD and MMC disk names (migration-4.0)

SDMMC and MMC devicetree disk definitions now require `disk-name` so multiple devices can register. `"SD"` is the suggested SD default and `"SD2"` the suggested MMC default.

### SDHC and shell callbacks (migration-4.4)

Move `bus_4_bit_support`, `hs200_support`, and `hs400_support` from `sdhc_host_caps` to `sdhc_host_props`. `shell_set_bypass()` and `shell_bypass_cb_t` also gain a user-data pointer.

### Stream Flash erasure (4.1.0)

`stream_flash_erase_page()` is deprecated; use `flash_area_erase()` or `flash_erase()`. Erasing storage directly can destroy Stream Flash data and is appropriate only when Stream Flash is not configured to erase or when removing data before or after its use of the area.

## Timers, counters, watchdogs, and other devices

### 64-bit counter ticks (migration-4.4)

Drivers implementing `get_value_64` must select `CONFIG_COUNTER_SUPPORTS_64BITS_TICKS`, and applications must select `CONFIG_COUNTER_64BITS_TICKS` before using that API.

### AXP MFD and nRF ETR configuration (migration-4.3)

The combined `MFD_AXP192_AXP2101` symbol is removed; select `MFD_AXP192` or `MFD_AXP2101` for the corresponding device. The nRF ETR driver moved under debug: rename `NRF_ETR*` symbols to `DEBUG_NRF_ETR*` and explicitly enable `DEBUG_DRIVER`.

### Driver API introspection (4.1.0)

`DEVICE_API_IS` tests whether a device implements a particular API. Shell completion uses it to offer only compatible devices where a command expects one.

### i.MX GPT run mode (migration-4.4)

`nxp,imx-gpt` now defaults to `run-mode = "restart"`, which resets the counter at Compare Channel 1 alarms. Set `run-mode = "free-run";` to preserve continuous pre-4.4 counting.

### Infineon and interrupt-controller names (migration-4.4)

Infineon CAT1 Kconfigs and bindings lose `CAT1`, for example `CONFIG_*_INFINEON_CAT1` becomes `CONFIG_*_INFINEON` and `infineon,cat1-adc` becomes `infineon,adc`; its CYW43xx HCI UART symbols become `CONFIG_BT_HCI_UART_INFINEON` and `infineon,bt-hci-uart`. Rename `swerv,pic` to `cdns,swerv-pic`.

### nPM1300 to nPM13xx migration (migration-4.2)

GPIO, LED, MFD, regulator, sensor-charger, and watchdog headers, APIs, defines, and Kconfigs move from `NPM1300`/`npm1300` names to `NPM13XX`/`npm13xx`. Regulator rail GPIOs no longer reference a GPIO controller: replace a value such as `enable-gpios = <&pmic_gpios 3 GPIO_ACTIVE_LOW>;` with `enable-gpio-config = <3 GPIO_ACTIVE_LOW>;`.

### NXP includes and system timers (migration-4.4)

NXP ARM DTSI includes move into family directories, for example `<nxp/nxp_rt1060.dtsi>` becomes `<nxp/imxrt/nxp_rt1060.dtsi>`. Boards using `CONFIG_MCUX_LPTMR_TIMER` must select the timer with `/chosen/zephyr,system-timer = &lptmr0`.

### NXP LPTMR filtering (migration-4.4)

`nxp,lptmr` no longer treats `prescale-glitch-filter = <0>` as bypass; use the boolean `prescale-glitch-filter-bypass;`, and keep active filter values in the range 0–15. In time-counter mode the divisor is `2^(N + 1)`; in pulse-counter mode a change is recognized after `2^N` rising edges and zero is not a valid active filter.

### Out-of-tree driver API declarations (migration-4.1)

Driver APIs now live in iterable sections for runtime validation. Out-of-tree implementations of upstream driver classes must declare their API with `DEVICE_API()`.

### SCMI call controls (4.3.0)

`ARM_SCMI_CHAN_SEM_TIMEOUT_USEC` configures the SCMI channel semaphore timeout, and `scmi_send_message()` gains an argument selecting polling. Callers should use `scmi_status_to_errno()` directly to translate returned command status.

### System-timer low-power companion (migration-4.4)

Out-of-tree Cortex-M timer code should replace `z_cms_lptim_hook_on_lpm_entry/exit` with `z_sys_clock_lpm_enter/exit`, the `CONFIG_CORTEX_M_SYSTICK_LPM_TIMER_*` family with `CONFIG_SYSTEM_TIMER_LPM_COMPANION_*`, and `/chosen/zephyr,cortex-m-idle-timer` with `/chosen/zephyr,system-timer-companion`.

### Watchdog startup (migration-4.4)

`CONFIG_WDT_DISABLE_AT_BOOT=n` no longer means a watchdog is automatically configured and running. Applications must configure it explicitly; the STM32, Raspberry Pi Pico, and TI `*_INITIAL_TIMEOUT` options used for the old behavior are removed.
