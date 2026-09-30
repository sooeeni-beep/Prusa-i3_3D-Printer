# Prusa i3 Custom — Hardware & Marlin Configuration

This repository documents the actual hardware of this custom Prusa i3 and the Marlin configuration used for it.

> **Firmware baseline:** Marlin 2.1.2.6 configuration files  
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

No configuration changes have been committed yet. This section will be updated together with each firmware configuration change.
