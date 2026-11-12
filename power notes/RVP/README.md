# Reverse-polarity protection (RVP)

## What this block does

A 1S LiPo is a polarized pack with a JST connector that is easy to reverse if someone forces it or uses a non-standard lead. This block lets current through only when the pack is the right way around, and disconnects `BATT+` from the rest of the board if it is not.

It is a high-side P-channel MOSFET used as a near-ideal diode, not a series Schottky. The extra drop is tens of millivolts instead of ~0.4 V, which matters on a 3.0–4.2 V cell.

## How it is used here

The pack (1S, 2000 mAh, with its own protection PCM) plugs into J1. Pack positive is `BATT+`. After Q1 it becomes `BATT_NET`, which feeds the BQ24074 BAT pins and the battery-monitor divider. Pack negative is board GND.

The cell's built-in PCM already handles over-charge, over-discharge, and short-circuit at the pack. Q1 does not replace that PCM; it only protects the board from a reversed connector.

**Check polarity on J1 before plugging in a pack.** JST-PH housings are keyed, but lead colors and pinouts are not always consistent across vendors.

## Parts

| Ref | Value / MPN | Why it is here |
|---|---|---|
| J1 | JST-PH 2-pin (typical PH 2.0 mm) | Battery connector. Pin 1 = `BATT+`, pin 2 = GND. Confirm the footprint against the pack lead. |
| Q1 | AO3401A, 30 V P-channel MOSFET, SOT-23 | Reverse-polarity switch. VGS(max) is ±12 V, so a 4.2 V cell does not need a gate Zener. |
| R1 | 100 kΩ | Holds the gate at GND so VGS = −Vbat whenever the pack is inserted correctly. |

Datasheet: [AO3401A (Alpha & Omega)](https://www.aosmd.com/res/datasheets/AO3401A.pdf).

## Pin-by-pin (Q1 AO3401A, SOT-23)

| Pin | Name | Connects to |
|---|---|---|
| 1 | Gate | R1 to GND |
| 2 | Source | `BATT_NET` (protected pack rail) |
| 3 | Drain | `BATT+` (J1 pack positive) |

This is the standard high-side RVP orientation: drain toward the battery, source toward the load, gate to ground.

## How the FET turns on

A P-channel MOSFET has a body diode from drain to source (anode at the drain).

**Correct pack.** `BATT+` is at cell voltage. At the first instant the channel is off, so load current flows `BATT+` → drain → body diode → source → `BATT_NET`. As soon as `BATT_NET` rises, the source is at nearly Vbat and the gate is at 0 V, so

```text
Vgs = Vg - Vs = 0 - Vbat = -Vbat
```

That is about −3.0 V to −4.2 V, well past the −0.5 V to −1.3 V threshold. The channel turns on and shorts out the body diode. After that, current flows through the channel, not the diode.

**Reversed pack.** `BATT+` is at the cell negative and board GND is at the cell positive. The body diode is reverse-biased and Vgs does not turn the channel on, so `BATT_NET` stays disconnected.

No gate Zener is required. |Vgs| equals the cell voltage (≤ 4.2 V), and the AO3401A allows ±12 V.

## On-resistance and drop

From the AO3401A datasheet (not the coarser “~50 mΩ at −2.5 V” shorthand):

| Condition | RDS(on) typ | RDS(on) max |
|---|---|---|
| VGS = −10 V, ID = −4.0 A | 41 mΩ | 50 mΩ |
| VGS = −4.5 V, ID = −3.5 A | 47 mΩ | 60 mΩ |
| VGS = −2.5 V, ID = −2.5 A | 60 mΩ | 85 mΩ |

A full cell (Vgs ≈ −4.2 V) is close to the −4.5 V row. An empty cell (Vgs ≈ −3.0 V) is closer to the −2.5 V row. At a 2 A peak:

```text
Vdrop ≈ I × Rds(on)     →  0.09–0.17 V  (using typ 47 mΩ to max 85 mΩ)
P      ≈ I² × Rds(on)    →  0.19–0.34 W
```

That lines up with the design estimate of about 0.1–0.15 V and 0.2–0.3 W at 2 A. SOT-23 can live with a short 2 A peak; continuous 2 A at the high end of RDS(on) is warm, so keep copper around the source and drain pins.

## Gotchas

- Drain and source are easy to swap in a symbol. If they are backwards, the body diode still conducts in the forward direction and **does not protect against a reversed pack**.
- Do not put the FET in the ground lead. That would split the pack negative from board GND and break USB and charger return paths.
- R1 can stay 100 kΩ; gate leakage is tiny. A series gate resistor is optional and not used on V1.
- The pack PCM will disconnect the cell if it is over-discharged. `BATT_NET` will simply fall to 0 V; that is normal, not an RVP failure.
