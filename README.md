# Prusa i3 Custom — Hardware & Marlin Configuration

This repository documents the actual hardware of this custom Prusa i3 and the Marlin configuration used for it.

> **Firmware baseline:** Marlin 2.1.2.8 runtime with configuration files originally carrying `CONFIGURATION_H_VERSION 02010206`  
> **Controller:** Arduino Mega 2560 + RAMPS 1.4  
> **Power system:** 12 V DC

## Hardware inventory

| Subsystem | Installed hardware / known specification |
|---|---|
| Main MCU | Arduino Mega 2560 |
| Motion shield | RAMPS 1.4 |
| Supply | 12 V / 30 A switching PSU |
| Stepper drivers | TMC2208 StepStick modules, used in standalone STEP/DIR mode |
| RAMPS microstep jumpers | MS1, MS2 and MS3 physically fitted under X, Y, Z and E driver sockets |
| Effective TMC2208 STEP resolution | 1/16 microstep because MS1=HIGH and MS2=HIGH in standalone mode |
| Motors | NEMA 17, 4-wire; same motor family on all axes |
| Motor step angle | 1.8° / 200 full steps per revolution (STP-42D221-01 family) |
| X transmission | 2 mm pitch timing belt + 20-tooth motor pulley |
| Y transmission | 2 mm pitch timing belt + 20-tooth motor pulley |
| Z transmission | Two Z motors wired in series to one Z driver |
| Z leadscrews | 8 mm diameter, single-start, 2 mm pitch = 2 mm lead/rev |
| X endstop | Mechanical microswitch at MIN |
| Y endstop | Mechanical microswitch at MIN |
| Z endstop | Mechanical microswitch at MIN |
| Extruder | MK8 direct-drive, one extruder |
| Hotend cooling | 12 V fan + heatsink |
| Hotend thermistor | NTC 100K, Beta 3950 |
| Bed thermistor | NTC 100K, Beta 3950 |
| Heated bed | PCB Heated Bed MK2B Dual Power, operated at 12 V, approx. 210 × 210 mm physical size |
| Bed power stage | External heated-bed MOSFET/control board powered directly from the PSU; RAMPS provides the control signal |
| Desired maximum bed setpoint | 100 °C |
| Display | RepRapDiscount Smart Controller, character LCD with encoder |
| SD card | SD slot on the RepRapDiscount Smart Controller |
| Auto bed probe | None / not installed |

## Motion calculations

### X and Y

For a 1.8° motor:

- 200 full steps/revolution
- 1/16 external microstep resolution
- 20-tooth pulley
- 2 mm belt pitch

Travel per motor revolution:

```
20 teeth × 2 mm = 40 mm/rev
```

Steps per mm:

```
200 × 16 / 40 = 80 steps/mm
```

Therefore the baseline is:

- X = 80 steps/mm
- Y = 80 steps/mm

### Z

The Z leadscrew is single-start with 2 mm pitch, therefore lead = 2 mm/revolution.

```
200 × 16 / 2 = 1600 steps/mm
```

Therefore the baseline is:

- Z = 1600 steps/mm

The two Z motors are wired in series to one TMC2208 driver. This does not change the geometric steps/mm value.

### Extruder

The MK8 extruder E-steps must be calibrated on the actual machine. A final E-steps value should not be inferred solely from the MK8 name or from photographs because the effective hob diameter materially affects extrusion distance.

## TMC2208 mode

The installed TMC2208 modules are treated as **standalone STEP/DIR drivers**, not UART-controlled drivers.

For TMC2208 standalone mode the relevant configuration pins are MS1 and MS2:

| MS1 | MS2 | STEP input resolution |
|---|---|---:|
| LOW | LOW | 1/8 |
| HIGH | LOW | 1/2 |
| LOW | HIGH | 1/4 |
| HIGH | HIGH | 1/16 |

All RAMPS jumpers are fitted. Therefore MS1 and MS2 are HIGH and the firmware calculations use **1/16**.

The TMC2208 may internally interpolate motion to 256 microsteps; this does **not** change Marlin steps/mm.

Reference: Analog Devices / Trinamic TMC2208 datasheet.

## Temperature configuration basis

Both installed sensors are NTC 100K Beta 3950 thermistors. Marlin table **11** is the baseline selected for these sensors because Marlin describes it as a 100K, Beta 3950 thermistor table.

The heated-bed user setpoint is to be capped at 100 °C. Marlin separates the maximum target from the MAXTEMP shutdown margin using `BED_OVERSHOOT`. The configuration change log below records the exact values used.

Thermal runaway protection must remain enabled for both hotend and bed.

## Commissioning rules

Do not assume axis directions or endstop electrical polarity solely from the photographs. Before the first homing operation:

1. Check ambient hotend and bed temperatures for plausible readings.
2. Run `M119` with every endstop released, then press each switch individually and repeat `M119`.
3. Confirm X, Y and Z report the correct OPEN/TRIGGERED state.
4. Test very short axis moves away from physical limits before using `G28`.
5. Test heaters one at a time while watching the reported temperature.
6. Calibrate extruder E-steps before printing.
7. After verified calibration, save values with `M500`.

## Marlin configuration change log

### 2026-09-30 — Initial hardware-specific baseline

`Configuration.h` was changed from the raw Marlin defaults to a commissioning configuration for this printer.

#### Identity and controller

- `STRING_CONFIG_H_AUTHOR` changed from the default placeholder to `(sooeeni-beep, custom Prusa i3)`.
- `CUSTOM_MACHINE_NAME` enabled as `Prusa i3 Custom`.
- `MOTHERBOARD` remains `BOARD_RAMPS_14_EFB`, matching one hotend + fan + heated-bed control on RAMPS 1.4.
- Serial communication remains `SERIAL_PORT 0` at `250000` baud.

#### Stepper drivers

Changed all installed drivers from the raw A4988 defaults to standalone TMC2208:

```cpp
#define X_DRIVER_TYPE  TMC2208_STANDALONE
#define Y_DRIVER_TYPE  TMC2208_STANDALONE
#define Z_DRIVER_TYPE  TMC2208_STANDALONE
#define E0_DRIVER_TYPE TMC2208_STANDALONE
```

Reason: the physical modules are TMC2208 StepSticks with no UART wiring. RAMPS MS1/MS2/MS3 jumpers are fitted. In standalone operation MS1=HIGH and MS2=HIGH select a 1/16 STEP input resolution.

#### Motion calibration baseline

Changed:

```cpp
#define DEFAULT_AXIS_STEPS_PER_UNIT { 80, 80, 1600, 93 }
```

- X = 80 steps/mm — calculated from 200 steps/rev × 16 microsteps / (20 teeth × 2 mm).
- Y = 80 steps/mm — same transmission as X.
- Z = 1600 steps/mm — calculated from 200 steps/rev × 16 microsteps / 2 mm lead.
- E = 93 steps/mm — **provisional safe starting value only** for the MK8 direct extruder. It must be calibrated by commanding a known filament length and measuring the actual feed before printing.

The Z value replaces the raw Marlin value of 400 steps/mm, which was not compatible with the installed single-start 2 mm-lead screws at 1/16 STEP resolution.

#### Conservative commissioning motion limits

The raw Marlin motion limits were reduced for initial testing:

```cpp
#define DEFAULT_MAX_FEEDRATE          { 150, 150, 4, 25 }
#define DEFAULT_MAX_ACCELERATION      { 1000, 1000, 100, 5000 }
#define DEFAULT_ACCELERATION          800
#define DEFAULT_RETRACT_ACCELERATION  1000
#define DEFAULT_TRAVEL_ACCELERATION   1000
```

These are commissioning values, not final performance tuning values.

#### Thermistors

Both raw sensor selections were changed from table 1 to table 11:

```cpp
#define TEMP_SENSOR_0   11
#define TEMP_SENSOR_BED 11
```

Reason: the installed sensors are NTC 100K Beta 3950. Marlin 2.1.2.6 describes sensor table 11 as a 100K Beta-3950 thermistor table.

#### Temperature limits

Hotend:

```cpp
#define HEATER_0_MAXTEMP 260
#define HOTEND_OVERSHOOT 15
```

This gives a maximum normal target of 245 °C with Marlin's 15 °C overshoot reserve. This is a conservative initial limit for the installed MK8-style hotend until its exact heat-break/PTFE construction is confirmed.

Heated bed:

```cpp
#define BED_MAXTEMP   110
#define BED_OVERSHOOT 10
```

Marlin forbids a normal target above `MAXTEMP - OVERSHOOT`, so these values cap the user-set bed target at **100 °C**, while retaining a 10 °C fault margin for overshoot detection.

`THERMAL_PROTECTION_HOTENDS` and `THERMAL_PROTECTION_BED` remain enabled.

#### Endstops and probe

- X, Y and Z continue to use the MIN endstop connectors.
- The three endstop inversion values remain `false` as an initial NC-to-GND assumption.
- `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` was disabled because no bed probe / BLTouch is installed.
- Endstop polarity **must be verified with `M119` before the first `G28`**.

#### EEPROM

Enabled:

```cpp
#define EEPROM_SETTINGS
#define EEPROM_AUTO_INIT
```

This allows calibrated settings to be stored with `M500`.

After flashing this hardware-specific configuration for the first time, initialize the stored settings with:

```gcode
M502
M500
```

#### LCD and SD card

Enabled:

```cpp
#define SDSUPPORT
#define REPRAP_DISCOUNT_SMART_CONTROLLER
```

This matches the installed character RepRapDiscount Smart Controller and its onboard SD-card slot.

#### Geometry retained for commissioning

The following raw values were intentionally kept for the first commissioning build:

```cpp
#define X_BED_SIZE 200
#define Y_BED_SIZE 200
#define Z_MAX_POS  200
```

The physical heated bed is approximately 210 × 210 mm, but usable nozzle travel should be measured before increasing the printable area.

#### Configuration_adv.h

No hardware-specific changes were required in `Configuration_adv.h` for the first commissioning build. TMC2208 UART current control and diagnostics are intentionally not enabled because the installed drivers are being used in standalone mode.

## 2026-10-02 — Commissioning and thermal-control update

The first hardware commissioning pass was completed successfully.

### Verified hardware behavior

- X, Y and Z mechanical MIN endstops were rewired to use only RAMPS `S` and `-` (GND), with the +5 V pin unused.
- `M119` verified all three endstops as `open` when released and `TRIGGERED` when pressed.
- X homes left, Y homes rearward, and Z homes downward toward the bed.
- X/Y/Z motion was verified as smooth with no stepper chatter or abnormal heating.
- Four TMC2208 modules are installed in the correct orientation and operate stably in standalone STEP/DIR mode.
- Hotend and bed thermistors both report plausible ambient temperatures.
- Hotend heating was verified to 180 °C and bed heating was verified to 40 °C.
- The two electronics cooling fans (RAMPS/TMC2208 and external bed MOSFET cooling) were moved off D9 and connected to permanent 12 V so they run whenever the printer PSU is on.
- Measured supply at the external bed MOSFET input was approximately 12.03–12.10 V.

### Tuned hotend PID defaults

Hotend PID autotune produced:

```cpp
#define DEFAULT_KP  15.60
#define DEFAULT_KI   1.00
#define DEFAULT_KD  60.76
```

These values were first stored in EEPROM with `M301` + `M500`, then promoted into `Configuration.h` so they survive a future `M502`.

### Final heated-bed PID autotune

Bed PID autotune completed at 60 °C for 8 cycles. Final constants:

```cpp
#define DEFAULT_BED_KP 25.89
#define DEFAULT_BED_KI 0.81
#define DEFAULT_BED_KD 551.42
```

These values are now stored in `Configuration.h` as the firmware defaults for the installed MK2B bed and external MOSFET stage.

### Bed PID enabled

Bed control was changed from bang-bang to PID by enabling:

```cpp
#define PIDTEMPBED
```

The existing bed PID constants are only temporary defaults until the actual MK2B bed is autotuned after this firmware is flashed.

Recommended bed autotune after upload and EEPROM reset:

```gcode
M303 E-1 S60 C8
```

After autotune, record the returned bed Kp/Ki/Kd values, store them in EEPROM, and then promote them into `Configuration.h` as final defaults.

### Preheat safety alignment

The ABS preset bed temperature was reduced from 110 °C to 100 °C:

```cpp
#define PREHEAT_2_TEMP_BED 100
```

This aligns the preset with the intended maximum normal bed setpoint.

## First-flash validation sequence

After compiling and uploading Marlin, do **not** immediately home or heat the printer. Use this order:

1. Confirm the LCD starts and the SD card is detected.
2. Connect over USB at 250000 baud.
3. Check that hotend and bed both report plausible room temperature.
4. Send `M119` with all endstops released.
5. Press X, Y and Z switches individually and repeat `M119` to confirm each changes state correctly.
6. Make short manual movements on X/Y/Z while staying away from the endstops and confirm direction.
7. Only after direction and endstop logic are verified, run `G28`.
8. Test hotend heating at a low target first while continuously watching the reported temperature.
9. Test bed heating separately.
10. Calibrate E-steps, then store the final value with `M500`.
11. Tune hotend PID and save it. Bed PID is not enabled in this baseline.
