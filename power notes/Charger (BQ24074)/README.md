# Charger and power path — BQ24074

## What this IC does

The BQ24074 is a linear Li-ion charger with a USB-friendly input, and it is also the board's power-path switch. It makes `VSYS` from USB, from the battery, or from both at once, and it charges the cell with whatever input current the system is not using.

It does **not** report state of charge. PGOOD and CHG only tell you “input looks good” and “a charge cycle is in progress.” Charge percentage comes from the [battery-monitor divider](../Battery%20Monitor/README.md).

Datasheet: [TI BQ24074 (SLUS810N)](https://www.ti.com/lit/ds/symlink/bq24074.pdf). Figures below follow that datasheet. The related [BQ24075](https://www.ti.com/lit/ds/symlink/bq24075.pdf) is mentioned only as an alternative.

## How it is used here

`VBUS` enters IN. `BATT_NET` (after the reverse-polarity FET) enters BAT. OUT is `VSYS`, which feeds the 5 V boost.

Firmware does not run the charge algorithm. The IC does precharge, constant current, constant voltage, termination, and the safety timers by itself. Firmware only:

- holds `Charge_EN` (CE) low to allow charging, or high to stop it
- sets EN1 / EN2 to choose 100 mA, 500 mA, ILIM (wall), or suspend

## Pin-by-pin

| Pin | Name | Connection on this board | Notes |
|---|---|---|---|
| 1 | TS | R2, 10 kΩ to GND | The pin sources 75 µA. 10 kΩ to GND is the datasheet stand-in when no pack NTC is wired. That holds TS in the “temperature OK” window so charging is not disabled. |
| 2, 3 | BAT | Tied together to `BATT_NET`, C1 10 µF to GND | Place C1 at the pins. Confirm the symbol stacks or hides pin 3 on the same net. |
| 4 | CE | Active low. R3 10 kΩ pulldown plus `Charge_EN` GPIO | Charging is allowed when CE is low. The IC also has an internal ~285 kΩ pulldown, so it defaults to “charge on.” A GPIO is required only if firmware should be able to stop charging. Tying CE to GND is fine if that feature is dropped. Setting EN1 = EN2 = 1 also stops charging, but that is suspend: USB is disconnected and the whole system runs from the battery. Prefer CE if you only want to inhibit charge. |
| 5 | EN2 | `Charge_EN2` GPIO | Internal pulldowns on EN1 and EN2. No external default resistors on V1; DNF footprints are optional. |
| 6 | EN1 | `Charge_EN1` GPIO | See the truth table below. |
| 7 | PGOOD | Open-drain. D1 green LED via R4 470 Ω to `VSYS` | Low when the input is valid. Pulled to `VSYS`, not `+3.3V`, so the LED still works when the boost is off. |
| 8 | VSS | GND | |
| 9 | CHG | Open-drain. D2 red LED via R5 470 Ω to `VSYS` | Low only during the **first** charge cycle. A 2 Hz blink means a safety-timer fault (the cell never finished in time). |
| 10, 11 | OUT | Tied together as `VSYS`, C2 10 µF to GND | This is an **output**, not an input. Do not tie it to `VBUS`. With USB present, OUT is regulated to 4.4 V. On battery only, OUT is the cell voltage minus the internal FET drop. DPPM holds OUT near 4.3 V if the system load needs the battery to help. |
| 12 | ILIM | R6 1.5 kΩ to GND | Sets the input-current limit used in EN2-high “wall” mode. **Must be fitted even if you mostly use 500 mA USB mode.** If ILIM is left open, charging is disabled. |
| 13 | IN | `VBUS`, C3 1 µF to GND | Operating range 4.35–10.2 V. Overvoltage cutoff typ 10.5 V. Absolute maximum 28 V. Place C3 at the pin (25 V is enough; 50 V is fine). |
| 14 | TMR | Leave **open** | Open selects the default safety timers: 30 minutes precharge and 5 hours fast charge. If the cell never reaches full, the charger stops. That is what we want. |
| 15 | ITERM | Leave **open** | Open selects the default termination of 10% of the programmed charge current, about 49 mA at our 0.49 A charge setting. |
| 16 | ISET | R7 1.8 kΩ to GND | Sets fast-charge current. An optional ADC tap on this pin can watch live charge current (see below). |
| Pad | Thermal pad | GND, with vias | Required for heat. The IC is a linear charger and will get warm. Confirm the footprint's exposed pad (often pin 17) is actually tied to GND, even if the symbol hides it. |

## EN1 / EN2 truth table

Internal pulldowns mean both pins are 0 at reset and during ESP32 boot, until firmware drives them.

| EN2 | EN1 | Mode |
|---|---|---|
| 0 | 0 | 100 mA USB input limit (default at boot) |
| 0 | 1 | 500 mA USB input limit |
| 1 | 0 | ILIM (wall adapter), about 1.07 A here |
| 1 | 1 | Suspend: USB disconnected, system runs from the battery, charging stops |

A second of 100 mA charging at boot is harmless. External pulls are not required for V1.

## Design equations

Input current limit (ILIM mode):

```text
I_IN(max) = KILIM / RILIM
KILIM     = 1610 A·Ω     (datasheet)
RILIM     = 1.5 kΩ       (valid range 1.1 kΩ to 8 kΩ)

I_IN(max) = 1610 / 1500 ≈ 1.07 A
```

Fast-charge current:

```text
I_CHG = KISET / RISET
KISET = 890 A·Ω          (datasheet)
RISET = 1.8 kΩ           (valid range 590 Ω to 8.9 kΩ)

I_CHG = 890 / 1800 ≈ 0.49 A
```

On a 2000 mAh cell that is 0.25 C, a gentle V1 choice. The constant itself is only about ±10%, so a nearby 1% resistor is close enough.

ISET also carries a voltage proportional to the live charge current:

```text
V_ISET = (I_CHG / 400) × R_ISET
```

At 0.49 A and 1.8 kΩ, V_ISET is about 2.2 V. An ESP32 ADC on this pin is optional.

Linear-charger dissipation (datasheet power path):

```text
P = (V_IN - V_OUT) × (I_OUT + I_BAT) + (V_OUT - V_BAT) × I_BAT
```

When USB is in and OUT is 4.4 V, the first term is the 0.6 V IN-to-OUT drop times whatever current is going to the system and the battery. The second term is the remaining drop from 4.4 V down to the cell voltage times the charge current. This heat is why the thermal pad and GND vias matter. A 5 V, 0.5 A charge into a 3.6 V cell with a light system load is on the order of half a watt — verify with the equation and your actual currents rather than treating that as a rating.

## Why OUT is 4.4 V, not 5 V

The 0.6 V from USB 5 V down to 4.4 V is deliberate regulation, not random loss. It keeps `VSYS` inside a range that LiPo-tolerant parts can take, and it leaves headroom to charge a 4.2 V cell. The BQ24075 instead passes up to 5.5 V through on OUT and adds a SYSOFF pin that can disconnect the battery. V1 stays on the '74 and uses the boost EN pin as the power switch.

## Gotchas

- Pins 3 and 11 and the thermal pad are easy to leave floating if the symbol hides them. They must be on BAT, OUT, and GND respectively.
- ILIM cannot be left open.
- TMR and ITERM are intentionally open; do not pull them to GND or VSYS without reading those modes in the datasheet.
- Status LEDs are pulled to `VSYS` so they still work when `+3.3V` is off.
- CE low means “charge enabled.” A bug that leaves CE high will look like a dead charger.
