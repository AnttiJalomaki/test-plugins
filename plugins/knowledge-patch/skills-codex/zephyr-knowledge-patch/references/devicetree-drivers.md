# Devicetree, Drivers, and Hardware

Use this reference for Zephyr work in this topic area. Entries are organized by developer task rather than release order.

## 64-bit counter ticks (migration-4.4)

Drivers implementing `get_value_64` must select `CONFIG_COUNTER_SUPPORTS_64BITS_TICKS`, and applications must select `CONFIG_COUNTER_64BITS_TICKS` before using that API.

## ADC bindings and clocks (migration-4.3)

The Silabs IADC driver and compatible move from `iadc_gecko.c` and `silabs,gecko-iadc` to `adc_silabs_iadc.c` and `silabs,iadc`. STM32 ADC nodes now require `clock-names` matching `clocks`, using `adcx` for the register clock, `adc-ker` for the kernel source, and, where applicable, `adc-pre` for the RCC prescaler.

## ADC migrations (migration-4.4)

Rename `renesas,ra-adc` to `renesas,ra-adc12`, `CONFIG_ADC_MCUX_SAR_ADC` to `CONFIG_ADC_NXP_SAR_ADC`, and STM32 ADC `resolutions` to `st,adc-resolutions`. NXP SAR ADC nodes must add `zephyr,input-positive`, and supported SoCs should use `ADC_REF_VDD_1` rather than `ADC_REF_INTERNAL`.

## ADC sequence priority (4.4.0)

`adc_sequence.priority`, enabled with `CONFIG_ADC_SEQUENCE_PRIORITY`, lets ADC requests carry an explicit sequence priority.

## Additional removals and binding deprecations (4.3.0)

Devicetree's base `status` enum no longer accepts `ok`, STM32 LPTIM clock selection must move from `CONFIG_STM32_LPTIM_CLOCK_LSI`/`CONFIG_STM32_LPTIM_CLOCK_LSE` to Devicetree, and `maxim,ds3231` is deprecated in favor of `maxim,ds3231-rtc`.

## AXP MFD and nRF ETR configuration (migration-4.3)

The combined `MFD_AXP192_AXP2101` symbol is removed; select `MFD_AXP192` or `MFD_AXP2101` for the corresponding device. The nRF ETR driver moved under debug: rename `NRF_ETR*` symbols to `DEBUG_NRF_ETR*` and explicitly enable `DEBUG_DRIVER`.

## CAN API removals (4.1.0)

Replace `CAN_MAX_STD_ID` and `CAN_MAX_EXT_ID` with `CAN_STD_ID_MASK` and `CAN_EXT_ID_MASK`, and replace `can_get_min_bitrate()` and `can_get_max_bitrate()` with `can_get_bitrate_min()` and `can_get_bitrate_max()`. `can_calc_prescaler()` is removed without a listed replacement.

## CAN capacity configuration (migration-4.4)

The generic `CONFIG_CAN_MAX_FILTER`, `CONFIG_CAN_MAX_STD_ID_FILTER`, and `CONFIG_CAN_MAX_EXT_ID_FILTER` options are removed; configure the driver-specific limit, such as `CONFIG_CAN_MCUX_FLEXCAN_MAX_FILTERS` or the STM32 BXCAN/FDCAN standard and extended filter limits. FlexCAN's `CONFIG_CAN_MAX_MB` likewise moves to the per-instance `number-of-mb` Devicetree property.

## CAN driver settings (migration-4.4)

Rename `CONFIG_CAN_MCUX_MCAN` to `CONFIG_CAN_NXP_LPC_MCAN`. A `ti,tcan4x5x` node must set `ti,nwkrq-voltage-vio` to retain the former VIO default, while an `nxp,flexcan` `clk-source` now selects between the named `clksrc0` and `clksrc1` inputs.

## Clock-control bindings (migration-4.4)

Bouffalo Lab clock nodes move to the consolidated `bflb,flash-clk`, `bflb,pll`, and `bflb,root-clk` compatibles. Remove `resource-type`, `resource-instance`, and `resource-channel` from out-of-tree `infineon,peri-div` nodes.

## Connector and Nordic UART behavior (4.2.0)

Boards with Qwiic, Stemma, or Grove I2C connectors now expose the common `zephyr_i2c` devicetree label, allowing connectorized I2C shields to work across branding through `west build --shield`. The Nordic UART receiver mode that uses an extra timer is no longer deprecated because it is the reliable receive path without hardware flow control.

## Devicetree binding defaults (migration-4.4)

Bindings may no longer define defaults for `status`, `#address-cells`, or `#size-cells`; doing so is a build error. Put required values explicitly in the Devicetree source instead.

## Devicetree enum-array tests (migration-4.2)

`DT_ENUM_HAS_VALUE` and `DT_INST_ENUM_HAS_VALUE` now search every element of an array property rather than testing only its first element.

## Devicetree include and property style (migration-4.2)

Includes of Zephyr files moved out of `dts/common` must drop the `common/` prefix; Silicon Labs Series 2 SoC includes move under a superfamily directory such as `silabs/xg24/`. Local bindings must use hyphens rather than underscores in property names; `scripts/utils/migrate_bindings_style.py` performs the mechanical conversion.

## Devicetree iteration and nexus mappings (4.4.0)

`DT_FOREACH_REG`, `DT_FOREACH_REG_SEP`, `DT_FOREACH_REG_VARGS`, and `DT_FOREACH_REG_SEP_VARGS`, plus their instance-number variants, iterate register entries; `DT_CHILD_BY_UNIT_ADDR_INT` and `DT_INST_CHILD_BY_UNIT_ADDR_INT` select integer unit addresses. First-class `*-map` definitions add nexus-node and specifier-mapping support.

## Devicetree property enums (4.0.0)

Bindings can define `string-array` and `array` properties as enums, with accessors such as `DT_ENUM_IDX_BY_IDX`; generated strings now correctly escape quotes, backslashes, and newlines.

## Devicetree property spelling migrations (migration-4.1)

Several bindings replace underscore spellings with hyphenated names:

- Clock: `freqs_mhz`, `cg_reg`, and `pll_ctrl_reg` become `freqs-mhz`, `cg-reg`, and `pll-ctrl-reg`.
- Counter: `primary_source`, `secondary_source`, `filter_count`, and `filter_period` become `primary-source`, `secondary-source`, `filter-count`, and `filter-period`.
- CAN and display: `clock_div8`, `pclk_pol`, and `data_cmd-gpios` become `clock-div8`, `pclk-pol`, and `data-cmd-gpios`.
- DAC: `voltage_reference` and `power_down_mode` become `voltage-reference` and `power-down-mode`.
- GPIO: `pin_mask`, `pinmux_mask`, `vbatts_pins`, `bit_per_gpio`, `off_val`, and `on_val` become their corresponding hyphenated names.
- HW spinlock, I2C, I2S, and LED: use `num-locks`, `port-sel`, `fifo-depth`, and `max-curr-opt`.
- SDHC: `power_delay_ms` and `max_current_330` become `power-delay-ms` and `max-current-330`.
- Timer and USB: `ticks_us` and `phy_handle` become `ticks-us` and `phy-handle`.

## Devicetree register literals (migration-4.0)

`DT_REG_ADDR` and its variants now expand to unsigned literals, as do `DT_REG_SIZE` variants. Code that uses an address as a devicetree index must switch to the corresponding `DT_REG_ADDR_RAW` macro; there is no raw size variant.

## DMA userspace access (migration-4.3)

The DMA API no longer exposes userspace syscalls because their access and parameter verification could not be made safe. Userspace code can no longer invoke the DMA API through the former syscall surface.

## Driver API introspection (4.1.0)

`DEVICE_API_IS` tests whether a device implements a particular API. Shell completion uses it to offer only compatible devices where a command expects one.

## Driver compatible and power-domain migrations (migration-4.0)

Rename LiteX compatibles `litex,eth0` to `litex,liteeth` and `litex,uart0` to `litex,uart`. Microchip `microchip,mcp230xx`/`microchip,mcp23sxx` nodes must use a chip-specific compatible such as `microchip,mcp23017` and drop `ngpios`; open-drain MCP23x09/MCP23x18 outputs now expose their real semantics through `gpio_pin_set`.

Replace the singular `power-domain` property with `power-domains`. Providers declare `#power-domain-cells`, and consumers may name multiple entries with `power-domain-names`.

## EEPROM target access (migration-4.4)

I2C EEPROM target code should replace deprecated `eeprom_target_program()` with `eeprom_target_read_data()` and `eeprom_target_write_data()`, which take explicit offset and length arguments.

## ESP32-S3 LCD_CAM topology (migration-4.4)

`espressif,esp32-lcd-cam` now represents only the common peripheral and contains `espressif,esp32-lcd-cam-dvp` and `espressif,esp32-lcd-cam-mipi-dbi` child nodes. Move camera properties into the DVP child and point `zephyr,camera` at that child.

## FlexRAM, Ethos-U, and modem configuration (migration-4.2)

Include the NXP FlexRAM API from `<zephyr/drivers/misc/flexram/nxp_flexram.h>` and replace `memc_flexram_*` APIs and Kconfigs with `flexram_*`. Rename `CONFIG_ARM_ETHOS_U*` to `CONFIG_ETHOS_U*`, and replace removed `CONFIG_MODEM_CELLULAR_CMUX_MAX_FRAME_SIZE` with the separate `CONFIG_MODEM_CMUX_WORK_BUFFER_SIZE` and `CONFIG_MODEM_CMUX_MTU` settings.

## i.MX GPT run mode (migration-4.4)

`nxp,imx-gpt` now defaults to `run-mode = "restart"`, which resets the counter at Compare Channel 1 alarms. Set `run-mode = "free-run";` to preserve continuous pre-4.4 counting.

## I3C address-management API (4.0.0)

`i3c_ccc_do_setdasa()` now takes the dynamic address explicitly, `i3c_determine_default_addr()` is removed, and `attach_i3c_device()` no longer takes an address because the driver derives it from `i3c_device_desc`. Controllers may advertise SETAASA support with the `supports-setaasa` devicetree property.

## I3C target, RTIO, and controller handoff (4.1.0)

New I3C surfaces include `CONFIG_I3C_TARGET_BUFFER_MODE`, `CONFIG_I3C_RTIO`, `i3c_ibi_hj_response()`, `i3c_ccc_do_getacccr()`, and `i3c_device_controller_handoff()`. Initial controller bindings include `snps,designware-i3c` and `st,stm32-i3c`.

## Infineon and interrupt-controller names (migration-4.4)

Infineon CAT1 Kconfigs and bindings lose `CAT1`, for example `CONFIG_*_INFINEON_CAT1` becomes `CONFIG_*_INFINEON` and `infineon,cat1-adc` becomes `infineon,adc`; its CYW43xx HCI UART symbols become `CONFIG_BT_HCI_UART_INFINEON` and `infineon,bt-hci-uart`. Rename `swerv,pic` to `cdns,swerv-pic`.

## Mapped code partitions (migration-4.4)

Boards using `CONFIG_USE_DT_CODE_PARTITION` or `zephyr,code-partition` should migrate the selected node to `compatible = "zephyr,mapped-partition"`. Its unit address supplies the mapped address, nested partitions use `ranges` without `fixed-subpartitions`, and `CONFIG_FLASH_LOAD_OFFSET`/`CONFIG_FLASH_LOAD_SIZE` cannot be used with it.

## MDIO lifecycle (migration-4.4)

`mdio_bus_enable()` and `mdio_bus_disable()` are removed because MDIO drivers now manage bus state internally.

## Native PTY UART (migration-4.2)

`uart_native_posix` becomes `uart_native_pty`, with `zephyr,native-pty-uart`, `CONFIG_UART_NATIVE_PTY*`, and `CONFIG_UART_NATIVE_PTY_0` replacing the old POSIX names. Instantiate one devicetree node per UART; at runtime, `--<uart_name>_stdinout` connects an instance to standard input/output instead of a PTY.

## nPM1300 to nPM13xx migration (migration-4.2)

GPIO, LED, MFD, regulator, sensor-charger, and watchdog headers, APIs, defines, and Kconfigs move from `NPM1300`/`npm1300` names to `NPM13XX`/`npm13xx`. Regulator rail GPIOs no longer reference a GPIO controller: replace a value such as `enable-gpios = <&pmic_gpios 3 GPIO_ACTIVE_LOW>;` with `enable-gpio-config = <3 GPIO_ACTIVE_LOW>;`.

## nRF52/nRF53 internal regulators (migration-4.0)

The `CONFIG_SOC_DCDC_NRF52X*` and `CONFIG_SOC_DCDC_NRF53X*` options are deprecated in favor of regulator devicetree nodes. Set `regulator-initial-mode = <NRF5X_REG_MODE_DCDC>` on nodes such as `vregmain`/`vregradio`, and enable high-voltage nodes such as `reg0` or `vregh` with `status = "okay"`.

## nRF53 oscillator configuration (migration-4.0)

nRF53 LFXO/HFXO capacitor settings move from the deprecated `CONFIG_SOC_*LFXO*` and `CONFIG_SOC_*HFXO*` options to devicetree. Select external capacitors with `load-capacitors = "external"`; for internal capacitors also set `load-capacitance-picofarad` on LFXO or `load-capacitance-femtofarad` on HFXO.

## NXP binding migrations (migration-4.1)

Rename the following compatibles: `nxp,kinetis-adc12` to `nxp,adc12`, `nxp,imx-lpi2c` to `nxp,lpi2c`, `nxp,kinetis-mpu` to `nxp,sysmpu`, `nxp,kinetis-pinctrl` to `nxp,port-pinctrl`, `nxp,kinetis-pinmux` to `nxp,port-pinmux`, and `nxp,kinetis-ftm-pwm` to `nxp,ftm-pwm`. The MPU Kconfig rename is `CPU_HAS_NXP_MPU` to `CPU_HAS_NXP_SYSMPU`.

Also rename `nxp,kinetis-lpuart` to `nxp,lpuart`, `nxp,imx-lpspi` to `nxp,lpspi`, `nxp,kinetis-dspi` to `nxp,dspi`, `nxp,kinetis-rtc` to `nxp,rtc`, `nxp,kinetis-ftm` to `nxp,ftm`, and `nxp,kinetis-wdog32` to `nxp,wdog32`; the FTM binding also moves under the timer bindings.

## NXP EDMA discriminator (migration-4.4)

`CONFIG_DMA_MCUX_EDMA_V5` is removed now that EDMA v4 and v5 share one driver path; out-of-tree conditionals should use the unified `DMA_MCUX_EDMA_V4` handling.

## NXP FlexSPI NOR topology (migration-4.4)

An `nxp,imx-flexspi-nor` node is now a controller containing a `soc-nv-flash` child; move erase/write sizes and partitions to the child and add `ranges` to the controller. Point `zephyr,flash` at the child and `zephyr,flash-controller` at the controller.

## NXP I2S master clock direction (migration-4.2)

For `nxp,mcux-i2s`, set the new `mclk-output` devicetree property to make MCLK an output; `I2S_OPT_BIT_CLK_SLAVE` no longer controls MCLK direction.

## NXP includes and system timers (migration-4.4)

NXP ARM DTSI includes move into family directories, for example `<nxp/nxp_rt1060.dtsi>` becomes `<nxp/imxrt/nxp_rt1060.dtsi>`. Boards using `CONFIG_MCUX_LPTMR_TIMER` must select the timer with `/chosen/zephyr,system-timer = &lptmr0`.

## NXP LPTMR filtering (migration-4.4)

`nxp,lptmr` no longer treats `prescale-glitch-filter = <0>` as bypass; use the boolean `prescale-glitch-filter-bypass;`, and keep active filter values in the range 0–15. In time-counter mode the divisor is `2^(N + 1)`; in pulse-counter mode a change is recognized after `2^N` rising edges and zero is not a valid active filter.

## NXP USDHC card detection (migration-4.0)

Without a configured card-detect method, NXP USDHC now assumes a card is present. Add `detect-cd` to the active USDHC node to keep using the peripheral's internal card-detect signal.

## Other compatible migrations (migration-4.1)

Rename `ti,ads114s0x-gpio` to `ti,ads1x4s0x-gpio`, `renesas,ra8-pwm` to `renesas,ra-pwm`, and `zephyr,gpio-steppers` to `zephyr,gpio-stepper`. Sensor compatibles `we,wsen-pads`, `we,wsen-pdus`, `we,wsen-tids`, and `invensense,icp10125` become `we,wsen-pads-2511020213301`, `we,wsen-pdus-25131308XXXXX`, `we,wsen-tids-2521020222501`, and `invensense,icp101xx`, respectively.

Silabs serial bindings split into `silabs,usart-uart` for Series 2 and `silabs,gecko-usart` for Series 0/1. SPI compatibles `silabs,gecko-spi-usart` and `silabs,gecko-spi-eusart` become `silabs,usart-spi` and `silabs,eusart-spi`; the deprecated `eth_mcux` driver is removed.

## Out-of-tree driver API declarations (migration-4.1)

Driver APIs now live in iterable sections for runtime validation. Out-of-tree implementations of upstream driver classes must declare their API with `DEVICE_API()`.

## Partition macros (migration-4.4)

The `FIXED_PARTITION_*` macro family is deprecated in favor of corresponding `PARTITION_*` names, such as `PARTITION_ID`, `PARTITION_OFFSET`, `PARTITION_DEVICE`, and `PARTITION_BY_NODE`; the replacements also support `zephyr,mapped-partition`.

## PSRAM refresh configuration (migration-4.4)

`st,stm32-xspi-psram` and `st,stm32-ospi-psram` nodes must set `st,refresh` in memory-clock cycles; their hard-coded defaults are gone.

## QSPI and radio bindings (migration-4.4)

An STM32 QSPI node using `dual-flash` must add `ssht-enable` to retain sample shifting, which now defaults off. Rename `generic-fem-two-ctrl-pins`, `gpio-radio-coex`, and `tx-high-power-supported` to their `radio-`-prefixed forms.

## Raspberry Pi and STM32 configuration (migration-4.1)

Rename `CONFIG_SOC_SERIES_RP2XXX` to `CONFIG_SOC_SERIES_RP2040`. STM32 `st,adc-sequencer` and `st,adc-clock-source` properties now take strings instead of integer values.

## Renesas RA flash naming (migration-4.2)

Rename `CONFIG_RA_FLASH_HP` to `CONFIG_SOC_FLASH_RENESAS_RA_HP` and `CONFIG_FLASH_RA_WRITE_PROTECT` to `CONFIG_FLASH_RENESAS_RA_HP_WRITE_PROTECT`; `CONFIG_DUAL_BANK_MODE` is removed. The generic `renesas,ra-nv-flash` binding is split into `renesas,ra-nv-code-flash` and `renesas,ra-nv-data-flash`.

## SCMI call controls (4.3.0)

`ARM_SCMI_CHAN_SEM_TIMEOUT_USEC` configures the SCMI channel semaphore timeout, and `scmi_send_message()` gains an argument selecting polling. Callers should use `scmi_status_to_errno()` directly to translate returned command status.

## SCMI firmware interface (4.0.0)

Zephyr gains initial Arm SCMI support for subsets of clock and pin-control commands over shared-memory and mailbox transports.

## SD and MMC disk names (migration-4.0)

SDMMC and MMC devicetree disk definitions now require `disk-name` so multiple devices can register. `"SD"` is the suggested SD default and `"SD2"` the suggested MMC default.

## SDHC and shell callbacks (migration-4.4)

Move `bus_4_bit_support`, `hs200_support`, and `hs400_support` from `sdhc_host_caps` to `sdhc_host_props`. `shell_set_bypass()` and `shell_bypass_cb_t` also gain a user-data pointer.

## Sensor bindings (migration-4.0)

MCP9808 nodes move from `microchip,mcp9808` to the generic `jedec,jc-42.4-temp` compatible. Current-sense amplifiers replace `sense-resistor-micro-ohms` with `sense-resistor-milli-ohms`, and `sense-gain-mult`/`sense-gain-div` are now limited to `UINT16_MAX`; `nxp,kinetis-acmp` properties should drop their deprecated `nxp,` name prefix.

## Sensor compatible migrations (migration-4.2)

Use `liteon,ltrf216a` instead of `ltr,f216a`, `ti,tmp11x`/`ti,tmp11x-eeprom` instead of the TMP116-only compatibles, and a pressure-specific `meas,ms5837-30ba` or `meas,ms5837-02ba` instead of `meas,ms5837`. The WSEN ITDS compatible becomes `we,wsen-itds-2533020201601`.

## Signal-only IPM callbacks (4.4.0)

Mailbox-backed IPM now supports signal-only messages; callbacks must accept a `NULL` payload when the mailbox supplies no data buffer.

## Silabs RAIL and power-state configuration (migration-4.3)

Rename `CONFIG_RAIL_PA_CURVE_HEADER`, `CONFIG_RAIL_PA_CURVE_TYPES_HEADER`, and `CONFIG_RAIL_PA_ENABLE_CALIBRATION` to their `CONFIG_SILABS_SISDK_RAIL_*` forms. Series 2 SoCs remove the separate `em3` state and now choose EM2 or EM3 automatically from peripheral oscillator requests.

## Silabs Series 2 pin control (migration-4.1)

Series 2 devices use the new `silabs,dbus-pinctrl` driver, with signal macros from a SoC binding header and GPIO electrical properties expressed in devicetree groups:

```devicetree
group0 {
    pins = <I2C0_SDA_PD2>, <I2C0_SCL_PD3>;
    drive-open-drain;
    bias-pull-up;
};
```

## SoC Kconfig renames (migration-4.4)

Nordic series symbols such as `CONFIG_SOC_SERIES_NRF52X`, `CONFIG_SOC_SERIES_NRF54HX`, and `CONFIG_SOC_SERIES_NRF91X` lose their trailing `X`; SiFive Freedom symbols drop `SIFIVE_FREEDOM` from names such as `CONFIG_SOC_SERIES_SIFIVE_FREEDOM_FU700`; and `CONFIG_SOC_SERIES_CH32V00X` becomes `CONFIG_SOC_SERIES_QINGKE_V2C`.

## SPI chip-select timing (migration-4.3)

`SPI_CS_CONTROL_INIT*`, `SPI_CONFIG_DT*`, and `SPI_DT_SPEC_GET*` no longer accept a delay argument. Put peripheral chip-select timing in devicetree with `spi-cs-setup-delay-ns` and `spi-cs-hold-delay-ns`.

## SPI NOR deep power-down (migration-4.0)

`CONFIG_SPI_NOR_IDLE_IN_DPD` is removed. Enable device runtime power management for the SPI NOR device and tune the replacement behavior with `CONFIG_SPI_NOR_ACTIVE_DWELL_MS`.

## STM32 ADC clock source (migration-4.0)

Every STM32 ADC selecting the asynchronous source with `st,adc-clock-source` must now also define its domain clock explicitly with the `clock` property.

## STM32 clock configuration (migration-4.3)

`CONFIG_CLOCK_STM32_HSE_CLOCK` is no longer user-configurable on STM32 MPU platforms; it is derived from `clock-frequency` on an enabled `&clk_hse`. The removed `st,stm32f1-rcc` and `st,stm32f3-rcc` bindings no longer provide ADC prescaler properties, so supply the prescaler as another ADC clock.

## STM32 DCMI and external memories (migration-4.2)

STM32 DCMI sensor-interface settings move to endpoint-based `video-interfaces.yaml` bindings, and `capture-rate` is replaced by `video_set_frmival()`. STM32 xSPI/oSPI/qSPI memory nodes now separate mapping and capacity: for a 512-Mbit device use `reg = <0>;` and `size = <DT_SIZE_M(512)>;`, with the mapping address supplied by the SoC controller node.

## STM32 PLL bindings (migration-4.4)

STM32F2/F4/F7 PLL compatibles merge into `st,stm32fx-pll-clock`, with `div-divq`/`div-divr` renamed to `post-div-q`/`post-div-r`; define a post-divider whenever the corresponding `div-q`/`div-r` is used. STM32L4 PLLSAI likewise moves to `st,stm32l4-pll-clock` and `post-div-r`.

## STM32 power and tightly coupled memory (migration-4.4)

STM32 power-supply Kconfigs are removed in favor of the `st,stm32h7-pwr`, `st,stm32h7rs-pwr`, and `st,stm32-dualreg-pwr` bindings. Replace `/chosen/zephyr,ccm` and `__ccm_*_section` with `/chosen/zephyr,dtcm` and `__dtcm_*_section`.

## STM32 SPI and interrupt sizing (migration-4.4)

`CONFIG_SPI_STM32_USE_HW_SS` is removed: `cs-gpios` or `st,soft-nss` selects Soft NSS, and their absence selects Hard NSS. SPI pins now default to `very-high-speed` slew and may need an overlay to reduce power, while auto-derived `CONFIG_NUM_IRQS` may need an explicit larger value for IRQs used only through `IRQ_CONNECT()`.

## STM32N6 security state (migration-4.4)

STM32N6 projects must now explicitly select either `CONFIG_TRUSTED_EXECUTION_SECURE` or `CONFIG_TRUSTED_EXECUTION_NON_SECURE` according to the state in which Zephyr executes.

## STM32U5 OTG HS PHY clock (migration-4.3)

`st,stm32u5-otghs-phy` nodes must set the new `clock-reference` property to select SYSCFG OTG HS PHY CLKSEL consistently with the RCC OTG HS kernel-clock selection.

## STM32U5 voltage scaling (4.4.0)

The `voltage-scale` property on `st,stm32u5-pwr` selects the voltage scale manually, notably allowing USB operation at lower system clock frequencies.

## u-blox GNSS (migration-4.0)

The purported M10 driver is now identified as M8-only: change compatibles to `u-blox,m8` and Kconfig to `CONFIG_GNSS_U_BLOX_M8`. The `gnss_set_periodic_config` and `gnss_get_periodic_config` APIs are removed.

## UART readiness and LiteX Kconfig (migration-4.0)

Treat `uart_irq_tx_ready()` as ready when its return value is greater than zero, not only when it equals one; the value is now a lower bound on bytes accepted by `uart_fifo_fill()`. Rename `CONFIG_UART_LITEUART` to `CONFIG_UART_LITEX`.

