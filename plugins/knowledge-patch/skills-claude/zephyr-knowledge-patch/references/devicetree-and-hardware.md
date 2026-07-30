# Devicetree and Hardware Description

Devicetree syntax and macros, bindings, board hardware descriptions, clocks, power domains, and SoC integration.

## Bindings, properties, and generated macros

### Additional removals and binding deprecations (4.3.0)

Devicetree's base `status` enum no longer accepts `ok`, STM32 LPTIM clock selection must move from `CONFIG_STM32_LPTIM_CLOCK_LSI`/`CONFIG_STM32_LPTIM_CLOCK_LSE` to Devicetree, and `maxim,ds3231` is deprecated in favor of `maxim,ds3231-rtc`.

### Devicetree binding defaults (migration-4.4)

Bindings may no longer define defaults for `status`, `#address-cells`, or `#size-cells`; doing so is a build error. Put required values explicitly in the Devicetree source instead.

### Devicetree enum-array tests (migration-4.2)

`DT_ENUM_HAS_VALUE` and `DT_INST_ENUM_HAS_VALUE` now search every element of an array property rather than testing only its first element.

### Devicetree include and property style (migration-4.2)

Includes of Zephyr files moved out of `dts/common` must drop the `common/` prefix; Silicon Labs Series 2 SoC includes move under a superfamily directory such as `silabs/xg24/`. Local bindings must use hyphens rather than underscores in property names; `scripts/utils/migrate_bindings_style.py` performs the mechanical conversion.

### Devicetree iteration and nexus mappings (4.4.0)

`DT_FOREACH_REG`, `DT_FOREACH_REG_SEP`, `DT_FOREACH_REG_VARGS`, and `DT_FOREACH_REG_SEP_VARGS`, plus their instance-number variants, iterate register entries; `DT_CHILD_BY_UNIT_ADDR_INT` and `DT_INST_CHILD_BY_UNIT_ADDR_INT` select integer unit addresses. First-class `*-map` definitions add nexus-node and specifier-mapping support.

### Devicetree property enums (4.0.0)

Bindings can define `string-array` and `array` properties as enums, with accessors such as `DT_ENUM_IDX_BY_IDX`; generated strings now correctly escape quotes, backslashes, and newlines.

### Devicetree property spelling migrations (migration-4.1)

Several bindings replace underscore spellings with hyphenated names:

- Clock: `freqs_mhz`, `cg_reg`, and `pll_ctrl_reg` become `freqs-mhz`, `cg-reg`, and `pll-ctrl-reg`.
- Counter: `primary_source`, `secondary_source`, `filter_count`, and `filter_period` become `primary-source`, `secondary-source`, `filter-count`, and `filter-period`.
- CAN and display: `clock_div8`, `pclk_pol`, and `data_cmd-gpios` become `clock-div8`, `pclk-pol`, and `data-cmd-gpios`.
- DAC: `voltage_reference` and `power_down_mode` become `voltage-reference` and `power-down-mode`.
- GPIO: `pin_mask`, `pinmux_mask`, `vbatts_pins`, `bit_per_gpio`, `off_val`, and `on_val` become their corresponding hyphenated names.
- HW spinlock, I2C, I2S, and LED: use `num-locks`, `port-sel`, `fifo-depth`, and `max-curr-opt`.
- SDHC: `power_delay_ms` and `max_current_330` become `power-delay-ms` and `max-current-330`.
- Timer and USB: `ticks_us` and `phy_handle` become `ticks-us` and `phy-handle`.

### Devicetree register literals (migration-4.0)

`DT_REG_ADDR` and its variants now expand to unsigned literals, as do `DT_REG_SIZE` variants. Code that uses an address as a devicetree index must switch to the corresponding `DT_REG_ADDR_RAW` macro; there is no raw size variant.

### NXP binding migrations (migration-4.1)

Rename the following compatibles: `nxp,kinetis-adc12` to `nxp,adc12`, `nxp,imx-lpi2c` to `nxp,lpi2c`, `nxp,kinetis-mpu` to `nxp,sysmpu`, `nxp,kinetis-pinctrl` to `nxp,port-pinctrl`, `nxp,kinetis-pinmux` to `nxp,port-pinmux`, and `nxp,kinetis-ftm-pwm` to `nxp,ftm-pwm`. The MPU Kconfig rename is `CPU_HAS_NXP_MPU` to `CPU_HAS_NXP_SYSMPU`.

Also rename `nxp,kinetis-lpuart` to `nxp,lpuart`, `nxp,imx-lpspi` to `nxp,lpspi`, `nxp,kinetis-dspi` to `nxp,dspi`, `nxp,kinetis-rtc` to `nxp,rtc`, `nxp,kinetis-ftm` to `nxp,ftm`, and `nxp,kinetis-wdog32` to `nxp,wdog32`; the FTM binding also moves under the timer bindings.

### Other compatible migrations (migration-4.1)

Rename `ti,ads114s0x-gpio` to `ti,ads1x4s0x-gpio`, `renesas,ra8-pwm` to `renesas,ra-pwm`, and `zephyr,gpio-steppers` to `zephyr,gpio-stepper`. Sensor compatibles `we,wsen-pads`, `we,wsen-pdus`, `we,wsen-tids`, and `invensense,icp10125` become `we,wsen-pads-2511020213301`, `we,wsen-pdus-25131308XXXXX`, `we,wsen-tids-2521020222501`, and `invensense,icp101xx`, respectively.

Silabs serial bindings split into `silabs,usart-uart` for Series 2 and `silabs,gecko-usart` for Series 0/1. SPI compatibles `silabs,gecko-spi-usart` and `silabs,gecko-spi-eusart` become `silabs,usart-spi` and `silabs,eusart-spi`; the deprecated `eth_mcux` driver is removed.

### QSPI and radio bindings (migration-4.4)

An STM32 QSPI node using `dual-flash` must add `ssht-enable` to retain sample shifting, which now defaults off. Rename `generic-fem-two-ctrl-pins`, `gpio-radio-coex`, and `tx-high-power-supported` to their `radio-`-prefixed forms.

### RISC-V Devicetree ownership (migration-4.4)

`CONFIG_RISCV` now requires a `riscv` Devicetree node, whose `riscv,isa-base` and `riscv,isa-extensions` properties define the base ISA and extensions; `riscv,isa` is deprecated. SoC Kconfigs that encoded ISA/FPU choices, including the CV64A6 variants and AE350 `CONFIG_RV*`/FPU options, are removed or consolidated.

### RISC-V machine timer binding (migration-4.2)

Several vendor machine-timer compatibles are unified as `riscv,machine-timer`. Both MTIME and MTIMECMP addresses must be explicit, with matching required names:

```devicetree
reg = <0xd1000000 0x8>, <0xd1000008 0x8>;
reg-names = "mtime", "mtimecmp";
```

The CPU group's `timebase-frequency` property can now supply `CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC`.

### Sensor bindings (migration-4.0)

MCP9808 nodes move from `microchip,mcp9808` to the generic `jedec,jc-42.4-temp` compatible. Current-sense amplifiers replace `sense-resistor-micro-ohms` with `sense-resistor-milli-ohms`, and `sense-gain-mult`/`sense-gain-div` are now limited to `UINT16_MAX`; `nxp,kinetis-acmp` properties should drop their deprecated `nxp,` name prefix.

### Sensor compatible migrations (migration-4.2)

Use `liteon,ltrf216a` instead of `ltr,f216a`, `ti,tmp11x`/`ti,tmp11x-eeprom` instead of the TMP116-only compatibles, and a pressure-specific `meas,ms5837-30ba` or `meas,ms5837-02ba` instead of `meas,ms5837`. The WSEN ITDS compatible becomes `we,wsen-itds-2533020201601`.

## Clocks, power, and regulators

### ADC bindings and clocks (migration-4.3)

The Silabs IADC driver and compatible move from `iadc_gecko.c` and `silabs,gecko-iadc` to `adc_silabs_iadc.c` and `silabs,iadc`. STM32 ADC nodes now require `clock-names` matching `clocks`, using `adcx` for the register clock, `adc-ker` for the kernel source, and, where applicable, `adc-pre` for the RCC prescaler.

### Clock-control bindings (migration-4.4)

Bouffalo Lab clock nodes move to the consolidated `bflb,flash-clk`, `bflb,pll`, and `bflb,root-clk` compatibles. Remove `resource-type`, `resource-instance`, and `resource-channel` from out-of-tree `infineon,peri-div` nodes.

### Driver compatible and power-domain migrations (migration-4.0)

Rename LiteX compatibles `litex,eth0` to `litex,liteeth` and `litex,uart0` to `litex,uart`. Microchip `microchip,mcp230xx`/`microchip,mcp23sxx` nodes must use a chip-specific compatible such as `microchip,mcp23017` and drop `ngpios`; open-drain MCP23x09/MCP23x18 outputs now expose their real semantics through `gpio_pin_set`.

Replace the singular `power-domain` property with `power-domains`. Providers declare `#power-domain-cells`, and consumers may name multiple entries with `power-domain-names`.

### nRF52/nRF53 internal regulators (migration-4.0)

The `CONFIG_SOC_DCDC_NRF52X*` and `CONFIG_SOC_DCDC_NRF53X*` options are deprecated in favor of regulator devicetree nodes. Set `regulator-initial-mode = <NRF5X_REG_MODE_DCDC>` on nodes such as `vregmain`/`vregradio`, and enable high-voltage nodes such as `reg0` or `vregh` with `status = "okay"`.

### nRF53 oscillator configuration (migration-4.0)

nRF53 LFXO/HFXO capacitor settings move from the deprecated `CONFIG_SOC_*LFXO*` and `CONFIG_SOC_*HFXO*` options to devicetree. Select external capacitors with `load-capacitors = "external"`; for internal capacitors also set `load-capacitance-picofarad` on LFXO or `load-capacitance-femtofarad` on HFXO.

### NXP I2S master clock direction (migration-4.2)

For `nxp,mcux-i2s`, set the new `mclk-output` devicetree property to make MCLK an output; `I2S_OPT_BIT_CLK_SLAVE` no longer controls MCLK direction.

### STM32 ADC clock source (migration-4.0)

Every STM32 ADC selecting the asynchronous source with `st,adc-clock-source` must now also define its domain clock explicitly with the `clock` property.

### STM32 clock configuration (migration-4.3)

`CONFIG_CLOCK_STM32_HSE_CLOCK` is no longer user-configurable on STM32 MPU platforms; it is derived from `clock-frequency` on an enabled `&clk_hse`. The removed `st,stm32f1-rcc` and `st,stm32f3-rcc` bindings no longer provide ADC prescaler properties, so supply the prescaler as another ADC clock.

### STM32 PLL bindings (migration-4.4)

STM32F2/F4/F7 PLL compatibles merge into `st,stm32fx-pll-clock`, with `div-divq`/`div-divr` renamed to `post-div-q`/`post-div-r`; define a post-divider whenever the corresponding `div-q`/`div-r` is used. STM32L4 PLLSAI likewise moves to `st,stm32l4-pll-clock` and `post-div-r`.

### STM32U5 OTG HS PHY clock (migration-4.3)

`st,stm32u5-otghs-phy` nodes must set the new `clock-reference` property to select SYSCFG OTG HS PHY CLKSEL consistently with the RCC OTG HS kernel-clock selection.

### STM32U5 voltage scaling (4.4.0)

The `voltage-scale` property on `st,stm32u5-pwr` selects the voltage scale manually, notably allowing USB operation at lower system clock frequencies.

## Memory maps, partitions, and flash topology

### Mapped code partitions (migration-4.4)

Boards using `CONFIG_USE_DT_CODE_PARTITION` or `zephyr,code-partition` should migrate the selected node to `compatible = "zephyr,mapped-partition"`. Its unit address supplies the mapped address, nested partitions use `ranges` without `fixed-subpartitions`, and `CONFIG_FLASH_LOAD_OFFSET`/`CONFIG_FLASH_LOAD_SIZE` cannot be used with it.

### Partition macros (migration-4.4)

The `FIXED_PARTITION_*` macro family is deprecated in favor of corresponding `PARTITION_*` names, such as `PARTITION_ID`, `PARTITION_OFFSET`, `PARTITION_DEVICE`, and `PARTITION_BY_NODE`; the replacements also support `zephyr,mapped-partition`.

### STM32 power and tightly coupled memory (migration-4.4)

STM32 power-supply Kconfigs are removed in favor of the `st,stm32h7-pwr`, `st,stm32h7rs-pwr`, and `st,stm32-dualreg-pwr` bindings. Replace `/chosen/zephyr,ccm` and `__ccm_*_section` with `/chosen/zephyr,dtcm` and `__dtcm_*_section`.

## SoC and board hardware description

### Raspberry Pi and STM32 configuration (migration-4.1)

Rename `CONFIG_SOC_SERIES_RP2XXX` to `CONFIG_SOC_SERIES_RP2040`. STM32 `st,adc-sequencer` and `st,adc-clock-source` properties now take strings instead of integer values.

### SoC Kconfig renames (migration-4.4)

Nordic series symbols such as `CONFIG_SOC_SERIES_NRF52X`, `CONFIG_SOC_SERIES_NRF54HX`, and `CONFIG_SOC_SERIES_NRF91X` lose their trailing `X`; SiFive Freedom symbols drop `SIFIVE_FREEDOM` from names such as `CONFIG_SOC_SERIES_SIFIVE_FREEDOM_FU700`; and `CONFIG_SOC_SERIES_CH32V00X` becomes `CONFIG_SOC_SERIES_QINGKE_V2C`.

### STM32N6 security state (migration-4.4)

STM32N6 projects must now explicitly select either `CONFIG_TRUSTED_EXECUTION_SECURE` or `CONFIG_TRUSTED_EXECUTION_NON_SECURE` according to the state in which Zephyr executes.
