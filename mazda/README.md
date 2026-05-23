# Mazda

## Mazda EZ-6 / 6e (EU BEV, EPA1 platform)

Rebadged Changan Deepal SL03. EU model is pure BEV (the China REEV variant is not sold in Europe). Usable battery capacity is 68.8 kWh (LFP, 120 cells, 24 module thermistors).

### Vehicle details used to derive the profile

Reverse-engineered from a 2026 Mazda 6e (VIN prefix `LVRHDA...`) over a VLink BLE ELM327 adapter using Car Scanner Pro v2.1.25 captures and a follow-up live verification drive (190 sample ticks, mixed urban + brief highway burst). Every field was cross-checked against a value visible on the dashboard at capture time and validated against expected magnitudes under load.

| ABRP field | DID / PID | Decoded | Verification observation |
|---|---|---|---|
| `soc` | `22F1AF` on HPCM (RX `7EA`) | `(A<<8\|B)/10` % | drops monotonically with energy use; 1 % over ~3 km matches 68.8 kWh × consumption rate |
| `voltage` | `22F228` on BMS (RX `7A9`) | `(A<<8\|B)/10` V | range 370.6 V (sag under peak discharge) — 404.2 V (rest) |
| `current` | `22F229` on BMS | `((A<<8\|B)-6000)/10` A | range −231.6 A (regen) / +515.6 A (acceleration); positive = discharge / negative = regen — matches ABRP convention |
| `ext_temp` | `0146` Mode 01 | `A-40` °C | stable 22-24 °C vs dashboard |
| `vehicle_reported_speed` | `010D` Mode 01 | `A` km/h | 0-79 km/h tracked across drive |
| `odometer` | `22F1AE` on HPCM | `(A<<24\|B<<16\|C<<8\|D)/10` km | monotonic increase, tracks dashboard within rounding |

### Derived sanity check (not posted to ABRP — included here for verification)

`power = voltage × current / 1000` peaked at +191.55 kW under heavy acceleration, matching the Mazda 6e EU spec motor peak of 190 kW. Regen peak was −93 kW under hard braking. Both values lie cleanly inside the `current` minValue/maxValue envelope.

### What was reachable but intentionally omitted

The same vehicle exposes much more over OBD:

- 12 V auxiliary battery (`0142`, `(A<<8|B)/1000` V → ~14.25 V),
- BMS cell aggregates: min/max cell V (`22F250`/`F251`), min/max cell V position (`22F252`/`F253`),
- BMS cell min/max temperature aggregates (`22F254`/`F255`, 1-byte raw — scale not yet locked; vendor lookup table suspected),
- 120 individual cell voltages (`22F29F`-`F2EF`, `22F3A9`-`F3B0`, `22F3E1`-`F3FF`, each `(A<<8|B)` mV),
- 24 module temperatures (`22F1B0`-`F1B7` + `22F2F0`-`F2FF`, each `raw-40` = °C),
- HPCM pack-V mirror (`22F1BD`, ~10 V below `F228` — post-relay DC-link sense).

These are kept out to keep the poll cycle short. A separate diagnostic profile can expose them.

### Not on OBD for this car

- `is_charging` — ECU `0x700` DID `FD03` was a candidate (`0x00`=charging, `0x40`=idle) but live verification showed it flipping spuriously on door-open events. **Not reliable for telemetry — omitted from this profile.** A correct charging flag presumably lives on body-CAN behind the SVDC gateway.
- `batt_temp` aggregate — `22F254`/`F255` return single bytes whose raw values vary widely (8-112) while the displayed temperature stays stable (18-22 °C), implying a vendor lookup table rather than a linear formula.
- `soh` — dashboard shows SOCE 100 % but no live DID identified on the ECU set explored so far.
- `est_battery_range` — no OBD-reachable DID matches the dashboard value.
- HVAC state / setpoint, TPMS, cabin temp — live on body-CAN behind the SVDC gateway; not reachable via the OBD port without a mid-bus tap.
