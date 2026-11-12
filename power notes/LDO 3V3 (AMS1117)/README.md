# 3.3 V LDO — AMS1117-3.3

## What this block does

The AMS1117-3.3 is a simple linear regulator that turns the 5 V boost output into a 3.3 V rail for the ESP32-S3 and other digital logic.

It is not a low-noise analog regulator. Quiescent current is on the order of 5 mA, and the part is relatively noisy. That is acceptable for V1 digital logic. The scope's analog front end should later get its own quiet LDO (TPS7A20) on a separate branch off `+5V`.

Datasheets: official Advanced Monolithic Systems [ds1117.pdf](https://www.advanced-monolithic.com/pdf/ds1117.pdf) (the vendor site failed TLS / returned HTTP 500 from this environment). Working copy: [LCSC PDF](https://datasheet.lcsc.com/datasheet/pdf/7605756631f798031ba2cc3b28395648.pdf?productCode=C12087).

## How it is used here

U4 sits after the boost, not after `VSYS`. Input is `+5V`. Output is `+3.3V`. If the boost is off, this rail is off.

The path is `VSYS` (3.0–4.4 V) → 5 V boost → 3.3 V LDO. The extra up-then-down costs about 35% of the energy in this rail (the LDO burns the 1.7 V difference as heat). That is accepted for V1, especially when USB is plugged in. Rev B can run a 3.3 V buck-boost (TPS63001) straight from `VSYS` and only enable the 5 V boost when the analog side needs it.

## Pin-by-pin (U4 AMS1117-3.3, SOT-223)

| Pin / pad | Name | Connection |
|---|---|---|
| 1 | GND (ADJ on the adjustable version) | GND. On the fixed 3.3 V part this pin is ground. |
| 2 | VO | `+3.3V`, C10 22 µF tantalum to GND |
| 3 | VI | `+5V`, C9 10 µF to GND |
| Tab | GND / heatsink | GND. Use the tab and surrounding copper as the heatsink. |

Confirm the pin order against the exact footprint; some SOT-223 drawings number the tab as a fourth pin.

## Parts

| Ref | Value / MPN | Why it is here |
|---|---|---|
| U4 | AMS1117-3.3, SOT-223 | Fixed 3.3 V LDO, 800 mA-class part (far more than the digital load). Dropout is about 1.1 V typical, 1.3 V max in the AMS datasheet, so 5 V in is comfortable. |
| C9 | 10 µF | Input bypass on `+5V`. |
| C10 | TAJA226K010RNJ, 22 µF, 10 V tantalum (AVX TAJ, case A) | Output capacitor. The AMS1117 needs a few ohms of ESR to stay stable. A bare ceramic can oscillate. A tantalum of this size provides that ESR. A ceramic plus a 1 Ω series resistor is the alternative if you would rather not use tantalum. |

Capacitor datasheet: [Kyocera AVX TAJ series](https://datasheets.kyocera-avx.com/TAJ.pdf) (send a normal browser User-Agent; some clients get 406).

## Dissipation

Dropout is about 1.1 V, so at 300 mA:

```text
P ≈ (5.0 V - 3.3 V) × 0.30 A ≈ 0.51 W
```

That is acceptable on SOT-223 if the tab has copper. It will be warm to the touch at that load; it is not a thermal emergency. Quiescent current is about 5 mA, which is large by modern LDO standards and is always drawn whenever `+5V` is up.

## Gotchas

- Do not substitute a low-ESR ceramic on VO without the series 1 Ω (or a part the datasheet explicitly allows). Stability of the AMS1117 is the classic failure.
- This rail must not be used as the quiet analog supply for the scope front end.
- If `+5V` is missing, `+3.3V` will be missing. That is why the charger status LEDs are pulled to `VSYS` instead of 3.3 V.
