# dps310-nv

A driver for Infineon's DPS310, a digital barometric pressure sensor with
an I2C and an SPI interface, specified in Infineon's "DPS310 Digital
XENSIV Barometric Pressure Sensor for Portable Devices" data sheet, V1.2,
2020-10-15 (document reference IFX-sch1406115644540).  The driver
implements the `Barometer`, `Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv).  It reads the
coefficients from the part and compensates the readings with the formulas
of the data sheet's section 4.9.

## Install

```sh
novo pkg add dps310-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins, its CSB pin to the supply, which selects I2C, and
its SDO pin to the supply or nothing, which puts the part at 0x77.  The
program waits for the part to start, brings it up, reads the coefficients
once, and prints one line over RTT for each result, four a second.  At sea
level the pressure is about 101 325 000 millipascals.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dps310
use dps310.{ Dps310 }

// The part's address with the SDO pin high or open.
const ADDR = 0x77

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Dps310.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(15)
    if not part.begin()
        hal.uart.write("dps310: no answer\n")
        return
    let Some(cal) = part.calibration() else return
    loop
        if part.ready()
            match part.sample_with(cal)
                Some(s) =>
                    hal.uart.write("dps310")
                    field(" mC=", s.milli_c)
                    field(" mPa=", s.milli_pa)
                    hal.uart.write("\n")
                None =>
                    hal.uart.write("dps310: no result\n")
        else
            hal.timer.delay_ms(10)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 24,636 bytes | 9,196 bytes | 3,252 bytes |
| nRF54LM20 DK | 25,692 bytes | 9,444 bytes | 3,236 bytes |
| FRDM-MCXN947 | 26,268 bytes | 10,546 bytes | 2,748 bytes |

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04) and SPI
with a SPIM, on the pins `docs/hal/README.md` names under "The Nordic DKs'
buses", so all three rows are the image with the driver's reading path in it.
Before the Nordic buses landed the two DK rows were 13,088 and 13,936
bytes, the image without the reading path, because the board packages
answered for I2C and SPI as absent hardware.  A reply from the part over a
DK's bus is NOT VERIFIED until the driver rig carries one.  The compensation is binary64 arithmetic, which these
cores compute in software; the compiler's support library holds it, 7,926 bytes
of the FRDM-MCXN947 image.  The rest of each image is the runtime and
tables.  The driver's own code allocates nothing: every buffer is a
fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries a DPS310 yet.  The program
builds, and the driver's functions run over a double of the part that
follows the data sheet's register map and bus protocols.  A reading from a
part on a board has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The compensation functions are pure.  The driver performs
no input or output of its own and allocates nothing of its own.  The bus
functions of devices-nv build one short frame per transaction inside a
`system` block and release it when the call returns.  The module checks at
every tier, `embedded` included.  The program above declares `[hw]` for
the same reason.

## What it is

A barometer measures air pressure.  Air pressure at sea level is about
101 325 pascals (Pa), and falls by about 12 Pa for each metre of height
near the ground.  The package gives no altitude: that needs the pressure
at sea level where the board is, which changes with the weather.

The DPS310 measures pressure with a capacitive element and temperature
with a sensor in the ASIC or in the element.  Each result is a 24-bit
count, which the data sheet's formulas turn into pascals and degrees
Celsius with 9 coefficients that Infineon writes into the part at
production (Table 18).  The driver reads them from the 18 bytes at 0x10.

| Coefficient | Width | Bytes |
|---|---|---|
| c0 | 12 bits, signed | byte 0 and the high nibble of byte 1 |
| c1 | 12 bits, signed | the low nibble of byte 1 and byte 2 |
| c00 | 20 bits, signed | bytes 3 and 4 and the high nibble of byte 5 |
| c10 | 20 bits, signed | the low nibble of byte 5 and bytes 6 and 7 |
| c01, c11, c20, c21, c30 | 16 bits, signed | two bytes each, from byte 8 |

The count of a measurement is divided by a scale factor that depends on
its oversampling (Table 9).

| Oversampling | 1 | 2 | 4 | 8 | 16 | 32 | 64 | 128 |
|---|---|---|---|---|---|---|---|---|
| Code | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| Scale factor | 524288 | 1572864 | 3670016 | 7864320 | 253952 | 516096 | 1040384 | 2088960 |
| Time (Table 16) | 3.6 ms | 5.2 ms | 8.4 ms | 14.8 ms | 27.6 ms | 53.2 ms | 104.4 ms | 206.8 ms |

With the scaled counts Praw_sc and Traw_sc, the data sheet gives

```
Tcomp = c0 * 0.5 + c1 * Traw_sc
Pcomp = c00 + Praw_sc * (c10 + Praw_sc * (c20 + Praw_sc * c30))
      + Traw_sc * c01 + Traw_sc * Praw_sc * (c11 + Praw_sc * c21)
```

and the driver evaluates both in `Float`, binary64, in that order, and
rounds to a millidegree and a millipascal.  The traits round the pressure
to a pascal.  An oversampling above 8 times needs the matching shift bit
of CFG_REG; the driver sets P_SHIFT and T_SHIFT with the oversampling.

The part has three modes (section 4.1).

| Mode | What it does |
|---|---|
| Standby | no measurement; every register and the coefficients can be read, and the configuration is changed here |
| Command | one temperature or one pressure measurement on request, then standby |
| Background | pressure and temperature measured continuously at their own rates |

In background mode the rates are 1 to 128 results a second for each
quantity, and the two must fit in a second: the rate of temperature times
its measurement time plus the rate of pressure times its own is under one
second (section 4.4).  The driver checks this before it writes the rates.

The part can drive its SDO pin when a pressure, a temperature or a full
FIFO is ready.  `begin` turns on the pressure-ready interrupt where the
board wires an interrupt pin.  The pin is the address strap on I2C, so the
polarity must be the opposite of the strap: active high at 0x76 (SDO
low), active low at 0x77 (section 4.6).  The driver sets it from the
address.  On 4-wire SPI SDO is the data output and the part has no
interrupt line.

The part has two bus interfaces on the same pins.

| | I2C | SPI |
|---|---|---|
| Address | 0x77, or 0x76 with SDO low | chip select low |
| Several registers in one read | the register address advances by itself | the address advances by itself |
| Read | register, repeated start, data | control byte with bit 7 set (R/W), then data, with no dummy byte |
| Write | register, then the data | control byte with bit 7 clear, then one byte |
| Clock | 400 kHz | 4 MHz, mode 3 (the only mode, up to 10 MHz) |

Bringing CSB low once switches the part off I2C until the next power-on
(section 6).

## What the package contains

| Module | What is in it |
|---|---|
| `dps310` | The types `Dps310` (I2C) and `Dps310Spi` (SPI), `Dps310Cal` and `Dps310Sample`, the trait `Dps310Port`, the bus framing, the register access, the compensation, the modes, the settings, the FIFO and the bring-up |

## How to choose an entry point

A program on a board uses `Dps310` for a part on I2C and `Dps310Spi` for a
part on SPI.  Each is built by `attach` with the board's bus, its GPIO and
the wiring, as a board's device table builds it.  Both implement
`Barometer`, `Thermometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | resets and brings the part up, see below |
| `baro_start()`, `temp_start()` | in standby, start one pressure or one temperature measurement and answer the time to wait in microseconds; in background mode nothing is written and the answer is 0; -1 when the part did not answer |
| `baro_read()`, `temp_read()` | the latest pressure in pascals, and the temperature in millidegrees; each reads the coefficients first |
| `calibration()`, `sample_with(cal)` | the coefficients once, and a reading that uses them: one transaction fewer for each reading |
| `sample()` | the latest result as `Dps310Sample`: millidegrees and millipascals |
| `sample_once(cal)` | a temperature and then a pressure on request, waited for by polling the ready flags |
| `device_id()` | `Identifiable`: ID, 0x10, with REV_ID as `rev` |
| `set_oversampling(osr_p, osr_t)`, `set_rates(pm_rate, tmp_rate)` | the oversampling and the background rates, as codes 0 to 7, in standby |
| `sleep()`, `run()` | standby, and continuous measurement |
| `ready()`, `clear_irq()` | the result-ready flags of MEAS_CFG, and INT_STS, which clears on read |
| `fifo_begin()`, `fifo_end()`, `fifo_take(cal, press, temp)` | the FIFO, which is off until `fifo_begin` |
| `irq_level()` | the level of the interrupt pin, -1 for a wiring with none |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` checks ID, resets the part with the soft-reset command and waits
for SENSOR_RDY and COEF_RDY, reads the coefficients once to prove them
readable, sets TMP_EXT to the sensor the coefficients were made for, sets
8 times oversampling and 4 results a second for both quantities and the
interrupt, applies the temperature workaround below, measures a
temperature on request so the pressure compensation has one, and starts
continuous measurement.  It answers `false` for a part that does not
answer 0x10 and for a register that does not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Dps310Port` over its bus and calls the module's functions,
which take the port: `dps310.sample_read(port)`, `dps310.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.  The compensation is
`cal_of`, `temperature_c` and `pressure_pa`, which need no bus.

### The FIFO

The FIFO holds the last 32 results of background mode, pressure and
temperature mixed in the order they were taken (section 4.8).  It is an
option: `begin` leaves it off.  `fifo_begin` empties it, turns it on and,
where an interrupt pin is wired, swaps the pressure-ready interrupt for
the FIFO-full one.  `fifo_take` drains up to 32 words into two lists, the
pressures in millipascals and the temperatures in millidegrees, oldest
first, and answers how many words it took.

The least significant bit of a word says what it is, 1 for a pressure and
0 for a temperature, and stays in the count.  A pressure is compensated
with the latest temperature before it in the batch, or with the first one
in the batch when it comes before any; a pressure in a batch with no
temperature is taken out and dropped.  The FIFO stops recording when it is
full and does not overwrite, so a program drains it before it fills: at 4
results a second of each kind, in under four seconds.

## The rules a user needs

1. `begin` has to run before a reading.  The data registers read 0 before
   the first measurement, and 0x800000 once the FIFO has run dry; a
   reading of either answers `None`.
2. The time `baro_start` answers is the time to wait before the read, plus
   a fifth over Table 16's typical figure.  A read that comes sooner
   answers the previous result.
3. A pressure measurement on request uses the temperature last measured.
   In standby, call `temp_start`, wait, then `baro_start`, wait, or call
   `sample_once`.
4. Read the coefficients once with `calibration()` and use `sample_with` in
   a loop.  `baro_read` and `temp_read` read them again each time.
5. The counts and the two oversampling codes are read in one burst of
   eight registers, so a count is scaled with the oversampling it was
   taken at.
6. A change of oversampling or rate is made in standby, and `set_oversampling`
   and `set_rates` refuse (write nothing, answer `false`) otherwise.  The
   Linux driver found that MEAS_CFG does not take a new mode unless it
   was written 0 first, and every mode change here goes through standby.
7. The coefficients are valid for one temperature sensor.  `begin` copies
   TMP_COEF_SRCE into TMP_EXT, as Zephyr's, Infineon's and the Linux
   driver do; a program that changes TMP_CFG with `set_reg` has to keep
   that bit.
8. Some parts read about 60 °C when the temperature is about 20 °C.
   Infineon's fix is a write sequence to the undocumented registers 0x0E,
   0x0F and 0x62.  Zephyr and Infineon apply it to every part; the Linux
   driver reads the undocumented register 0x32 first and applies it only
   when bit 1 is clear, and so does this one.  The data sheet does not
   mention the registers.
9. A command mode (a measurement on request) ends by itself, so the driver
   does not read MEAS_CTRL back after writing one, as it does for standby
   and the background modes.  `sample_once` waits by polling the ready
   flags, at most 20000 polls for each.
10. `device_id` answers `None` for a bus that no part drives: an SPI bus
    reads 0xFF, or 0x00 when held low.  The driver accepts ID 0x10, the
    one revision the data sheet lists.

## What is not included

- A self-test: the data sheet defines none.
- The SPI three-wire mode, and so an interrupt on SPI.
- The external temperature sensor choice other than the one the
  coefficients name, and an altitude.
- The interrupt on a temperature result (INT_TMP) and the FIFO-full
  interrupt on SPI; `reg` and `set_reg` reach CFG_REG.
- The async form of the traits, which an `async fn` cannot take as a value
  type.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- bmp390-nv drives Bosch's BMP390, a barometer with the same traits, the
  same bus conventions in other places and a compensation that needs no
  scale factors.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus, with the start-up bits that read not ready
for the first polls, the flags that reading the results clears, the FIFO
that returns 0x800000 when empty, and the undocumented check register.
It checks the framing of every entry point, the coefficient decoding at
the ends of every width, Table 9's factors, the conversion times, the
modes and their refusals, the settings and the rate budget, the FIFO, the
workaround, the bring-up and its refusals, and the two types over the
host's board handles.  A second port refuses a chosen write or read of a
chosen register.  The suite covers every line of `src/`.

The compensation is checked against vectors worked by hand.  The data
sheet gives the formulas and no numeric example, and Infineon's reference
driver (`DpsClass::calcTemp` and `calcPressure` in the Arduino library)
gives the same formulas, so the suite's comment derives the first vector
term by term: with c0 = 175, c1 = -2000, c00 = 126653, c10 = -55000,
c01 = -1700, c11 = -1000, c20 = 10000, c21 = -1200 and c30 = -2000, a
temperature count of 245760 and a pressure count of 3932160, both at 8
times, are Traw_sc = 1/32 and Praw_sc = 1/2, and give 25 °C and 101324.875
Pa exactly.  Three more vectors come from a separate transcription of the
formulas in binary64, and cover other oversamplings and negative counts.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, Zephyr's `drivers/sensor/infineon/dps310`, Infineon's
`DPS310-Pressure-Sensor` Arduino library and the Linux kernel's
`drivers/iio/pressure/dps310.c`.  Embassy and esp-hal hold no driver for
the part.  Each fix kept is named in `src/dps310.nv` where it is kept.

## Licence

Apache-2.0.
