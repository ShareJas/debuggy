# Battery monitor (voltage divider)

## What this block does

V1 estimates pack charge from cell voltage, using a resistor divider into an ESP32-S3 ADC pin. The result is a coarse battery icon (a few bars), not a percentage accurate to a few points.

A fuel gauge is not required for that. The BQ24074 also does not help here: it only reports power-good and charging. A dedicated gauge (MAX17048, or BQ27441-G1 if you want current sense) is a Rev B upgrade.

## How it is used here

`BATT_NET` (the protected pack rail, after Q1) is divided 2:1 and filtered, then named `BATT_MON`. Firmware reads it on an ADC1 pin, converts the voltage back to cell voltage, and maps that to bars. Readings should be taken when the load is light, because a LiPo's terminal voltage sags under load and sits in a flat region in the middle of the discharge curve.

## Parts

| Ref | Value | Why it is here |
|---|---|---|
| R8 | 1 MΩ, from `BATT_NET` to `BATT_MON` | Top of the divider. 1 MΩ keeps idle drain very small. |
| R9 | 1 MΩ, from `BATT_MON` to GND | Bottom of the divider. Equal values give a divide-by-two. |
| C4 | 100 nF, `BATT_MON` to GND | Holds the node steady so the ADC samples the capacitor, not a 1 MΩ source. |

No extra datasheet; these are standard passives. The ADC behavior is in the [ESP32-S3 datasheet](https://documentation.espressif.com/esp32-s3_datasheet_en.pdf).

## Voltages and ADC setup

```text
V_BATT_MON = V_BATT_NET × R9 / (R8 + R9) = V_BATT_NET / 2
```

| Cell voltage | `BATT_MON` |
|---|---|
| 4.2 V (full) | 2.1 V |
| 3.0 V (empty) | 1.5 V |

Both are inside the ESP32-S3 ADC range when the pad is set to **11 dB attenuation**.

Use an **ADC1** pin, GPIO1–GPIO10. ADC2 cannot be used reliably while Wi-Fi is on.

Idle current in the divider:

```text
I ≈ 4.2 V / (1 MΩ + 1 MΩ) ≈ 2.1 µA
```

(That is why the resistors were raised from 100 kΩ, which would have been about 21 µA, including while the product is switched off.)

## Firmware guidance

- Sample slowly, around 1 Hz, and average 8–16 readings. The 1 MΩ / 100 nF time constant is 50 ms; a burst of back-to-back samples does not help.
- Expect a few percent of offset. Calibrate once against a known meter on `BATT_NET` / TP3.
- Suggested bar thresholds (cell voltage, after you convert the ADC reading):

| Cell voltage | UI |
|---|---|
| above 3.9 V | Full |
| 3.7–3.9 V | Mid |
| 3.5–3.7 V | Low |
| below 3.4 V | Shed loads (turn off the 5 V boost extras / DC output per the load-shedding plan) |

These are coarse, voltage-only thresholds, not a state-of-charge model.

## Why not a fuel gauge on V1

A LiPo voltage curve is flat through the middle and depends on load, temperature, and where you are in the charge cycle. A gauge such as the MAX17048 integrates a model (and, on parts like the BQ27441-G1, also measures current through a sense resistor) and reports a stable percentage. That is nicer, but it is another IC, another library part, and I2C work. For a 3- or 4-bar icon, the divider is enough.

Rev B candidates, if you outgrow this:

- [MAX17048](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX17048-MAX17049.pdf) — no sense resistor, I2C, optional low-SOC alert pin
- BQ27441-G1 — more accurate, needs a 10 mΩ sense resistor in the pack ground return (awkward with USB ground)

## Gotchas

- The divider is always connected to `BATT_NET`, so it draws ~2 µA even when the boost is off. That is negligible next to LiPo self-discharge.
- Do not hang this divider on `BATT+` before Q1; a reversed pack would put a negative voltage on the ADC pin.
- Do not use ADC2 if Wi-Fi is on.
- High-value resistors need C4. Without it, the ADC's sample-and-hold will pull the node around and the reading will jump.
