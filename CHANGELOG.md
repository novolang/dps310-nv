# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Dps310Port` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Dps310Port` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Dps310Port` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Dps310Port for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `i2c_setup`, `irq_level_of`, `reg_read`, `reg_write`, `reg_update`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Dps310Port` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 19,968 bytes before and 19,904 after on the read-out (nrf52-qemu: 8,400 and 8,400).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.2), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 20,068 B before and 20,012 B after; nrf54lm20-dk 21,292 B before and 21,232 B after; frdm-mcxn947 21,456 B before and 21,416 B after; nrf52-qemu 8,404 B before and 8,412 B after.

## 0.1.0

The first release: a driver for Infineon's DPS310 digital barometric
pressure sensor on an I2C or an SPI bus.

- `Dps310` (I2C) and `Dps310Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Barometer`, `Thermometer` and
  `Identifiable`, with their own methods for the coefficients, a reading in
  millidegrees and millipascals, a measurement on request, the
  oversampling, the background rates, the data-ready interrupt, the FIFO
  and register access.
- The register access as functions over `Dps310Port`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.  An SPI read carries no dummy byte, and an SPI
  part has no interrupt line.
- The compensation of the data sheet's section 4.9 in binary64: the 9
  coefficients from the 18 bytes at 0x10, Table 9's scale factor for each
  oversampling, the temperature and the pressure.
- Command mode and background mode, every mode change going through
  standby, and a check that the rates of the two quantities fit in a
  second.
- The shift bits an oversampling above 8 times needs, set with the
  oversampling.
- The interrupt polarity chosen from the I2C address, opposite to the
  strap on the SDO pin.
- The FIFO as an option: off after `begin`, on through `fifo_begin`, drained
  by `fifo_take` into pressures and temperatures.
- The temperature workaround of Infineon's library, applied the way the
  Linux driver applies it, when the undocumented check bit is clear.
- A reset with the soft-reset command, finished when SENSOR_RDY and
  COEF_RDY read 1.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a DPS310.
- The data sheet gives no numeric example of the compensation.  The suite's
  first vector is worked by hand from the formulas and the others come from
  a separate transcription of them.
- The data sheet read is V1.2 (2020-10-15), the revision that could be
  fetched from Infineon's community copy.  The temperature workaround is
  not in it; it comes from Infineon's library and the Linux driver.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  TWIM and SPIM drivers carry the reading path into the image: 24,636 and
  25,692 bytes of flash.  Before the Nordic buses landed the images were 13,088
  and 13,936 bytes, without the reading path.  A reply from the part over a
  DK's bus is NOT VERIFIED.
- The SPI three-wire mode and the temperature-ready interrupt are not
  implemented.
