# Debuggy

USB/battery-powered scope, logic analyzer, and multimeter (LDL instrument). KiCad project: `debuggy.kicad_pro`, schematics in `PCB/`.

- [Power notes](power%20notes/README.md): how power flows, per-block docs, datasheets

## Open items for next session (PCB design)

### Power (Rev A fixes)
- [ ] **F1 fuse:** change `1812L150` to `1812L150/16` (plain part is 8 V max). See [USB note](power%20notes/USB/README.md).
- [ ] **Shield:** add an optional 4.7 nF, 50 V footprint in parallel with R12 (can stay unpopulated).
- [ ] **Boost worst case:** check input current at a 3.0 V cell (I_in ≈ I_out × 5 / (3.0 × η)) against the inductor's 3.6 A guaranteed Isat and the TPS61023 switch current limit.
- [ ] **Low-battery cutoff:** pick firmware warn/shutdown thresholds on `BATT_MON` (~3.3–3.4 V under load); don't rely on the pack PCM (~2.5–3.0 V).
- [ ] **Battery-only mode:** confirm EN1 = EN2 = high (input suspend) GPIO control is routed, for running off battery with USB data connected.
- [ ] **Power switch:** fit the slide switch across J3 (`SYS_ON`).
- [ ] Info only: AO3401A RDS(on) at VGS = -2.5 V is 60 mΩ typ / 85 mΩ max (not ~50 mΩ). See [RVP note](power%20notes/RVP/README.md).

### Next blocks
- [ ] **Multimeter front end** (Kenneth, Tariq): next KiCad session.
- [ ] **Scope (Jack):** waiting on answers: 30 V input range?, 5 V / negative rail needs, sample rate (~40 MSPS pipeline ADC). Plan a TPS7A20 quiet LDO off `+5V` for the analog side.
- [ ] **ESP32-S3 WROOM:** native USB D+/D- through USBLC6, `BATT_MON` on an ADC1 pin, GPIOs for CE / EN1 / EN2.

### Rev B (not for this rev)
- TPS63001 3.3 V buck-boost straight from `VSYS`
- USB isolation (ADuM3160 + isolated DC-DC)
- Fuel gauge (MAX17048)
