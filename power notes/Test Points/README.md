# Test points and bring-up

## What this block does

Six test points give a safe place to meter the power rails during assembly, first power-on, and later debugging, without grabbing IC pins.

## Test point map

| TP | Net | What you should see |
|---|---|---|
| TP1 | `VBUS` | ~5 V only when USB is attached, measured after the fuse and TVS. 0 V otherwise. |
| TP2 | `VSYS` | ~4.4 V with USB attached. On battery only, about the cell voltage (3.0–4.2 V). Present even if the boost is off. |
| TP3 | `BATT_NET` | Cell voltage after the reverse-polarity FET. 0 V if no pack is installed, the pack PCM is open, or the pack is reversed. |
| TP4 | `+5V` | 5.0 V when J3 is closed and `VSYS` is present. 0 V when J3 is open. |
| TP5 | `+3.3V` | 3.3 V whenever `+5V` is up. 0 V if the boost is off or the LDO has failed. |
| TP6 | GND | Board ground. Use this as the meter common for the other five. |

## Bring-up checklist

Do this in order. Do not plug in a LiPo as the first source.

1. **Visual.** Confirm Q1 orientation (drain to `BATT+`, source to `BATT_NET`, gate via R1 to GND), the inductor is actually an inductor (even if it is labeled R14), the BQ24074 thermal pad is grounded, and there are no solder bridges on USB VBUS/GND.
2. **Current-limited bench supply on `BATT+` first**, not USB. Set 3.7–4.0 V, current limit around 100–200 mA. Observe polarity. TP3 (`BATT_NET`) should rise to nearly the supply voltage (tens of millivolts less, from Q1). If TP3 stays at 0 V, stop and check Q1 and J1 polarity.
3. **Check `VSYS` (TP2)** with J3 still open. It should follow `BATT_NET`. The 5 V and 3.3 V rails should still be 0 V. The charger may be idle; that is expected with no USB.
4. **Close J3** (or jumper EN to `VSYS`). TP4 should be 5.0 V, TP5 should be 3.3 V. Input current should stay modest with no firmware load. If the 5 V rail will not start, check FB resistors, the inductor, and EN.
5. **Remove the bench lead, connect USB** (still no pack, or with the pack, as you prefer — first USB-only is safer). TP1 should be ~5 V. TP2 should go to ~4.4 V. With CE low and a pack attached, the red CHG LED should come on for a charge cycle and the green PGOOD LED should be on. If PGOOD is off, check the CC resistors, the fuse, and `VBUS`.
6. **With a pack and USB**, confirm `BATT_NET` is in the 3.0–4.2 V range and `BATT_MON` is half of that (1.5–2.1 V). Confirm the fuse does not trip during a wall-mode charge (ILIM ≈ 1.07 A). If you are still on a 1.1 A-hold fuse, this is where it may nuisance-trip.
7. **Open J3** with USB still connected. `+5V` and `+3.3V` should fall; `VBUS` and `VSYS` should remain, and the charge LEDs should still be able to light.
8. **Reversed-pack test only with a current-limited supply**, never with a real LiPo forced backwards. `BATT_NET` must stay at 0 V.

If anything is out of range, stop and fix that rail before moving down the chain. The [overview](../README.md) has the net table and the grounding warnings: this board's GND is USB GND, so treat bench-supply commons with care.
