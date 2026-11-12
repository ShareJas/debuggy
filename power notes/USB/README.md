# USB-C input

## What this block does

This is the 5 V input path into the charger, plus USB 2.0 data so the ESP32-S3 can be flashed from the same connector. There is **no USB Power Delivery**. The CC resistors ask only for 5 V.

## How it is used here

A 16-pin USB 2.0 Type-C receptacle (for example GCT USB4105-GF-A) brings in VBUS, GND, CC1, CC2, D+, D−, and the shell. After the CC handshake, VBUS goes through a resettable fuse and a TVS diode, then is named `VBUS` and feeds the BQ24074 IN pin. D+ / D− are ESD-clamped and routed as a 90 Ω differential pair to the ESP32-S3 USB pins (GPIO19 / GPIO20).

Because only the CC pulldowns are present, a PD charger cannot negotiate 9 V or 20 V. The BQ24074 also shuts off above 10.5 V and survives 28 V, so V1 omits a separate overvoltage-protection IC. A 0 Ω jumper footprint can be left in series if one is added later.

## Parts

| Ref | Value / MPN | Why it is here |
|---|---|---|
| J2 | USB-C receptacle, USB 2.0, 16P (e.g. GCT USB4105-GF-A) | Reversible 5 V + USB 2.0 data connector. Tie both VBUS pins together and both GND pins together; join D+ to D+ and D− to D− across the two rows. |
| R10, R11 | 5.1 kΩ, one on CC1 and one on CC2, each to GND | These are the USB-C Rd pulldowns. They tell the source to turn on 5 V. **Do not share one resistor between CC1 and CC2** — some chargers then never enable VBUS. |
| C5 | 1 µF, 50 V, at the connector | Local VBUS bypass. 50 V is oversized; 25 V is enough. Count this toward the 10 µF inrush budget. |
| F1 | Littelfuse 1812L150 family, 1.5 A hold | Resets after a short. Upsized from a 1.1 A-hold part (1812L110) because ILIM is about 1.07 A and a polyfuse's hold current falls when it is warm. See the voltage-rating note below. |
| D3 | SMAJ15A, unidirectional, after the fuse, VBUS to GND | Clamps surges below the BQ24074's 28 V absolute maximum. |
| U2 | USBLC6-2SC6 | ESD clamp on D+ and D−. Its VBUS reference pin goes to the **connector** `VBUS`, not to `+5V`, so the data lines stay clamped when the system is off. |
| R12 | 1 MΩ, shield to GND | Bleeds charge off the shell. Add an unpopulated 4.7 nF, 50 V footprint in parallel for a high-frequency path. Directly tying the shield to GND is also acceptable for V1. |

Datasheets:

- [GCT USB4105 drawing](https://gct.co/files/drawings/usb4105.pdf)
- [Littelfuse 1812L series](https://www.littelfuse.com/assetdocs/resettable-ptcs-1812l-datasheet?assetguid=ca5c80cb-504e-4a8a-8e74-0107520a1717)
- [Littelfuse SMAJ series](https://www.littelfuse.com/~/media/electronics/datasheets/tvs_diodes/littelfuse_tvs_diode_smaj_datasheet.pdf)
- [ST USBLC6-2](https://www.st.com/resource/en/datasheet/usblc6-2.pdf) (official ST URL; this environment could not complete a fetch — open it in a browser)

## Why not a 5 V TVS

A 5 V TVS (SMAJ5.0A class) would conduct hard if a bad 9–20 V source ever appeared on VBUS, and it would likely fail short or take the fuse with it. SMAJ15A stands off 15 V, so it stays off on a normal 5 V bus and still clamps below the charger's 28 V rating. The BQ24074 itself already disconnects the input at 10.5 V.

## Fuse current and voltage rating

F1 was first drawn as 1812L110 (1.1 A hold). ILIM is 1.5 kΩ, about 1.07 A, which is too close: polyfuse hold current is specified at 25 °C and derates as the part (and the nearby charger) warms up. A 1.5 A-hold device gives margin.

**Voltage rating, verify against the exact orderable part.** In the Littelfuse 1812L table:

| Part | Ihold | Vmax |
|---|---|---|
| 1812L150 | 1.50 A | **8 V** |
| 1812L150/16 | 1.50 A | **16 V** |

The design intent is 1.5 A hold and at least 16 V. The bare 1812L150 is only 8 V. Normal USB is 5 V, so 8 V can work, but a 9–15 V misplug would stress an 8 V fuse **before** the SMAJ15A starts to clamp (breakdown is about 16.7–18.5 V). Prefer **1812L150/16** (same 1812 footprint) if you want the 16 V rating.

## Capacitance and inrush

USB limits inrush by keeping downstream capacitance modest. On this board:

- C5, 1 µF at the connector
- C3, 1 µF at the BQ24074 IN pin

Keep the **total** on `VBUS` under 10 µF. Do not add a large electrolytic on this net.

## Optional CC current-capability detect

A USB-C source pulls each CC line up with a different Rp depending on what it can supply. With our 5.1 kΩ Rd, the CC voltage is approximately:

| Source advertisement | Typical Rp | Voltage on CC |
|---|---|---|
| Default USB (500 mA / 900 mA) | 56 kΩ | ~0.4 V |
| 1.5 A | 22 kΩ | ~0.9 V |
| 3.0 A | 10 kΩ | ~1.7 V |

Firmware can optionally ADC one CC line and pick EN1/EN2 to match. This is not required for V1; the charger can be left in 500 mA mode, or put in ILIM mode only when you know you are on a wall adapter.

## Gotchas

- Two separate 5.1 kΩ resistors, not one shared resistor, and not 5.1 kΩ to both pins from a single pad that can tombstone and leave both CC pins open.
- Route D+/D− as a short, matched 90 Ω pair to GPIO19 / GPIO20. Keep the USBLC6 right at the connector.
- U2's VBUS pin must follow connector `VBUS`, not the 5 V boost output.
- Shield: 1 MΩ alone is a slow bleed. Leave the 4.7 nF pads even if you do not populate them.
