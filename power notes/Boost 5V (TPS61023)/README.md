# 5 V boost — TPS61023

## What this block does

`VSYS` sits between about 3.0 V (empty cell, on battery) and 4.4 V (USB present, BQ24074 regulation). That is always **below** 5 V, so this rail only needs to step up. A buck-boost is unnecessary here and would be larger and more expensive.

The TPS61023 makes a regulated `+5V` rail for the AMS1117, and later for analog supplies and any 5 V accessories. Its EN pin is also the system on/off control.

Datasheet: [TI TPS61023](https://www.ti.com/lit/ds/symlink/tps61023.pdf). Confirm the feedback reference in that document before locking resistor values; the design below assumes **0.6 V**.

## How it is used here

U3 is a TPS61023DRLR (SOT-563, 6-pin). VIN comes from `VSYS`. EN is `SYS_ON`, pulled down by R13 and pulled up to `VSYS` when J3 is jumpered (or when a slide switch is fitted). VOUT is `+5V`.

## Pin-by-pin (U3 TPS61023DRLR)

| Pin | Name | Connection |
|---|---|---|
| 1 | FB | Divider from `+5V` to GND: R15 732 kΩ (top), R16 100 kΩ (bottom) |
| 2 | EN | `SYS_ON`. J3 (2-pin header) to `VSYS`. R13 100 kΩ to GND |
| 3 | VIN | `VSYS`, with C6 10 µF, 25 V, to GND, close to the pin |
| 4 | GND | Solid GND pour. Input and output capacitor grounds should meet here |
| 5 | SW | Inductor only. Keep this copper short and tight |
| 6 | VOUT | `+5V`, C7 and C8 each 22 µF, 25 V |

## Parts

| Ref | Value / MPN | Why it is here |
|---|---|---|
| U3 | TPS61023DRLR, SOT-563 | 5 V boost, ~1–1.5 A class output from a 3.0–4.4 V input. |
| C6 | 10 µF, 25 V (0805) | VIN bypass on `VSYS`. |
| L / R14 | Samsung CIGT201610LH1R0MNE, 1 µH ±20%, 2016 / 0806 | Power inductor. The schematic may still show a resistor symbol (R14) because that is how the SnapEDA / Ultra Librarian file came in. The **footprint is the inductor**. Treat it as L1 in the BOM discussion even if the reference is R14. |
| C7, C8 | 22 µF, 25 V, 0805 or larger, two in parallel | VOUT capacitance. 25 V in 0805 (or bigger) is intentional: smaller packages lose much of their capacitance at 5 V DC bias. |
| R15, R16 | 732 kΩ (top), 100 kΩ (bottom), 1% | Feedback divider. |
| R13 | 100 kΩ | EN pulldown so the boost is off unless J3 is closed. |
| J3 | 2-pin header | Temporary on/off. A slide switch can replace or sit across it. |

Inductor datasheet: [CIGT201610LH1R0MNE spec (Digi-Key host)](https://media.digikey.com/pdf/Data%20Sheets/Samsung%20PDFs/CIGT201610LH1R0MNE_Spec.pdf), [Samsung power inductor catalog](https://www.samsungsem.com/resources/file/global/support/product_catalog/Power_Inductor.pdf).

## Feedback equation

```text
VOUT = VREF × (1 + R15 / R16)
```

With VREF = 0.6 V (verify in the TPS61023 datasheet):

```text
VOUT = 0.6 × (1 + 732k / 100k) = 0.6 × 8.32 ≈ 5.00 V
```

732 kΩ is a standard 1% value. An equivalent pair with the same ratio, **220 kΩ top and 30 kΩ bottom**, also gives about 5.0 V and is a bit stiffer against FB leakage and noise. Either is fine:

```text
0.6 × (1 + 220k / 30k) = 0.6 × 8.333 ≈ 5.00 V
```

## Inductor current and why 3.6–3.9 A saturation is acceptable

Worst case is a low battery and a heavy 5 V load. For 1 A at 5 V from a 3.0 V input at 85% efficiency:

```text
P_OUT  = 5 V × 1 A = 5 W
P_IN   = 5 W / 0.85 ≈ 5.88 W
I_IN,avg ≈ 5.88 W / 3.0 V ≈ 2.0 A
```

If inductor ripple is 40% peak-to-peak, the peak is

```text
I_PEAK ≈ I_IN,avg + (0.40 × I_IN,avg) / 2 ≈ 2.0 + 0.4 = 2.4 A
```

Normal operation therefore sits around 2.0 A average and 2.4 A peak. The TPS61023 also has a **switch current limit of about 3.7 A**. That limit is not a typical load; it trips on a 5 V short, during startup while C7/C8 charge, or on a hard load step. In those moments the inductor really can see ~3.7 A.

If the inductor saturates well below that, its inductance collapses, current ramps faster than the chip can cut the cycle, and the switch can be stressed. So the inductor's saturation rating should be at or above that limit.

CIGT201610LH1R0MNE (from the Samsung spec sheet):

| Parameter | Typ | Max (or min, for ratings) |
|---|---|---|
| L | 1.0 µH | ±20% |
| DCR | 50 mΩ | 57 mΩ |
| Isat (30% L drop) | 3.9 A | 3.6 A guaranteed |
| Irms (40 °C rise) | 3.1 A | 2.8 A |

Irms of 2.8 A clears the ~2.4 A peak / ~2.0 A average running current. DCR is under 60 mΩ. The guaranteed Isat is 3.6 A, just under the ~3.7 A switch limit, with a typical of 3.9 A. Samsung defines saturation as a 30% inductance drop, which is a gradual roll-off, not a hard collapse, so a brief fault at the limit is acceptable for V1. Keep the SW node small and wide.

## On / off on EN

A pull-up alone (EN tied to `VSYS`) would leave the 5 V rail on whenever a battery or USB is present, with no way to turn the product off. J3 plus R13 is the V1 fix:

- J3 open: EN is low, the boost shuts down (datasheet shutdown current is very small; verify the exact microamp figure if you need a battery-life budget). Everything downstream, including the AMS1117, loses power.
- J3 closed: EN is at `VSYS`, the system boots from the battery alone.
- Charging still works with the boost off, because the BQ24074 is upstream of this switch.

## Gotchas

- The inductor may be labeled R14 on the schematic. Do not let that ship in the BOM as a resistor.
- Do not replace this with a buck-boost unless you also change the charger to the BQ24075 (or similar) and allow `VSYS` above 5 V.
- Place C6, the inductor, and C7/C8 as a tight loop. The SW node should not wander across the board.
- Ceramic output capacitors must keep enough capacitance at 5 V DC; 25 V, 0805 (or larger) parts are the safe choice.
