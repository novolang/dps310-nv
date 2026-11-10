# Changelog

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
