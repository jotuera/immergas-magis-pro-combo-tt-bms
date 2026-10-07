# Immergas Magis Pro / Combo (Audax) — Modbus BMS via ESPHome (M5 Atom)

Read and control an **Immergas Magis Pro / Combo** (Audax heat pump) over the **T-/T+ (BMS) Modbus RS485** bus using an **M5Stack Atom + ESPHome**, integrated into Home Assistant.

This is a community reverse-engineering project — Immergas does not publish the Modbus register map for this bus, so the map below was recovered by scanning + cross-referencing the Magis Combo installer manual, live logging of operating cycles, and the Samsung EHS/NASA documentation.

## Hardware

- **M5Stack Atom Lite** (ESP32) + isolated **RS485** transceiver (e.g. MAX3485 / M5 Isolated RS485 Unit)
- Wiring: Immergas **T+ / T-** → RS485 **A / B** (swap A/B if you get CRC errors)
- Bus: **Modbus RTU, slave 11, 9600 8N2**, 2-wire (the Immergas manual specifies 8N2; it also reads fine on 8N1 receivers, so either works)
- Tested on: **Magis Combo 9 Plus V2** (Audax 9, 9 kW, firmware D91 = 6.0) and **Magis Pro V2** (community-confirmed)
- The Atom is the Modbus **master**; the heat pump is slave 11.

## What it does

- **Reads**: flow/return/DHW/outdoor temperatures, **calculated flow setpoint (D04)**, compressor frequency, evaporator temperature, EEV position, outdoor fan speed, refrigerant-circuit temps, 3-way valve position, heat-pump/boiler/system status, fault code.
- **Controls**: operating mode (2000), DHW setpoint (2095), zone-1 heating/cooling limits (R04/R05/R12/R13), circulation pump min/max speed (A03/A04), zone-1 thermostat.
- **Decoded boiler fault** (114 Immergas anomaly codes), **calculated metrics** (Delta T, mean loop temp, thermal lift, heat-pump return − boiler flow difference).
- BMS connection toggle (start/stop polling).

## Requirements

- **ESPHome 2026.9 or newer.** From 2026.9 the frame spacing is set by the `modbus` hub (`turnaround_time`, default **600 ms**); `command_throttle` on the controllers no longer does anything. The config sets `turnaround_time: 80ms` so writes from Home Assistant stay fast. The deprecation warnings for `command_throttle` / `force_new_range` at compile time are harmless.

## ⚠️ Important caveats

- **Operating mode — whoever keeps asserting it wins.** An *emulated* zone panel on the D+/D- bus re-asserts its mode continuously and reverts any mode written via T-/T+ (register 2000). With a *real* Dominus panel the opposite was observed (firmware 6.0): once a mode has been written over T-/T+, the unit holds it as the BMS command and reverts mode changes made on the indoor-unit keypad or on the Dominus. With a T-/T+ master running, change the mode from Home Assistant.
- **Probes not fitted read `-32768` (0x8000).** E.g. register 3029 (boiler flow) on a heat-pump-only Magis Pro. All temperature sensors map this to *unknown* instead of showing −3276.8 °C.
- **Thermostat input 40-1/41 and register 2010.** On a Magis Combo with **firmware 6.0 and A31 = RPT**, an actively polling T-/T+ master does **not** block the zone thermostat — heat calls start the compressor normally. A user with **firmware 4.3 and A31 = RT** reports that an active master blocks the 40-1/41 input and that writing `2010 = 1` has no effect ([issue #1](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/1)). Both firmware and A31 differ between the two cases, so the cause is not pinned down yet — please report your D91 + A31 if you test this.

## Bus facts

- The unit answers **only Modbus function 0x03 (read holding registers)**. Functions 0x01 (coils), 0x02 (discrete inputs) and 0x04 (input registers) are rejected (illegal function).
- Register banks: **2xxx** (config/control), **3xxx** (sensors), **4xxx** (heat pump / refrigerant), **6xxx** (parameters / statuses). The whole range **0–19999** has been scanned; nothing answers above 6510.
- The map is fixed — entering the installer PIN on the indoor unit does **not** expose more registers. Heating-curve breakpoints (R02/R03) and most other menu parameters are **not** available on this bus.
- Immergas "PDU N" = holding register address N.

## Register map (known)

| Reg | Code | Description | Scale |
|---|---|---|---|
| 2000 | — | Operating mode (0=standby, 1=summer/DHW, 2=cooling, 3=winter) | RW |
| 2010 | — | Zone 1 thermostat / heating request (bit0) — see caveat above | RW |
| 2095 | D05 | DHW setpoint | ×0.1 °C, RW |
| 2100 | — | Fault code (0 = none, else E-code) | R |
| 3000 | D20 | System flow temperature | ×0.1 °C |
| 3001 | D08 | Heat pump return water temperature | ×0.1 °C |
| 3002 | D06 | Outdoor temperature | ×0.1 °C |
| 3003 | D04 | **Calculated flow setpoint** from the heating curve; 0 = no heat demand (earlier versions mislabelled it as R08) | ×0.1 °C |
| 3016 | D03 | Storage tank unit temperature | ×0.1 °C |
| 3028 | D22 | Generator 3-way valve (1 = DHW, 0 = CH) | 0/1 |
| 3029 | — | **Secondary generator (gas boiler) flow temperature** — reaches ~65 °C during DHW on gas; `-32768` on units without a boiler (earlier versions called it D23) | ×0.1 °C |
| 3054 | D14 | Circulator pump flow rate (0 while the pump is stopped) | l/h |
| 3056 | D24 | Chiller circuit liquid temperature | ×0.1 °C |
| 4202 | A11 | Outdoor Unit model | code (9 = 9 kW) |
| 4350 | R05 | Zone 1 minimum central heating | °C, RW |
| 4351 | R04 | Zone 1 maximum central heating | °C, RW |
| 4356 | R13 | Zone 1 maximum cooling | °C, RW |
| 4357 | R12 | Zone 1 minimum cooling | °C, RW |
| 4551 | D74 | **Evaporator (outdoor coil) temperature** — drops below outdoor temp while heating; can be negative | ×0.1 °C, signed |
| 4554 | — | Refrigerant hot-side temperature (candidate) | ×0.1 °C |
| 4557 | D77 | Electronic expansion valve (EEV) position (2000 = fully open at rest) | 0–2000 |
| 4558 | D71 | **Compressor frequency** (ramps ~15 → 30 Hz at start) | Hz |
| 4586 | — | **Compressor running** | 0/1 |
| 4587 | D76 | Outdoor unit fan speed (candidate; 450 while running) | rpm |
| 6000 | T05 | Central heating ignitions timer | RW |
| 6010 | A03 | Circulation pump minimum speed | %, RW |
| 6011 | A04 | Circulation pump maximum speed | %, RW |
| 6500 | D97 | Heat pump demand status: 300 = standby, 352 = heating, 322 = DHW (heat pump), 342 = cooling | 0–999 |
| 6501 | D98 | Thermal generator demand status: 200 = standby, 252 = heating | 0–999 |
| 6502 | D99 | System state — see below | 0–999 |
| 6506/6507 | D140/D141 | Internal RTC hour / minute (read-only, free-running) | h / min |

**D99 system state codes:** 0 / 82 = standby · 1 = cooling request · 6 = heating request · 8 = heating cycle (seen just before switching to DHW) · 41 = pump overrun after a CH cycle (~3 min) · 45 = end of DHW (~1 min) · 61 = DHW with solar boost · 62 = DHW on the gas boiler (Combo only) · 64 = DHW with the electric heater. DHW codes may depend on model and firmware — one unit on firmware 4.3 reports plain DHW as 61.

## 🔎 Unknown registers — contributions welcome!

Still unidentified: `2102, 4011, 4201, 4402, 4403, 4411, 4449, 4451, 4452, 4453, 4454, 4455, 4468, 4469, 4479, 4585, 4588, 4600, 6002, 6013, 6014, 6503, 6504, 6505, 6510` (exposed as `PDU <N>`, `Register <N> (RW 0-1)` or `Refrigerant circuit <N> (cand.)`). Most of them stay constant during short central-heating cycles at mild outdoor temperatures; they probably come alive during **DHW on the heat pump, defrost or high compressor load**. The six RW 0/1 registers do **not** change when the indoor-unit menu parameters U11, A35, A39 or P15 are toggled.

If you can correlate any of these with operating states, **please open an issue or PR** — that's how we finish the map.

## 🧪 Beta config — PDU scanner

`immergas-magis-pro-combo-tt-bms-beta.yaml` is the stable config **plus an active PDU scanner**:

1. Turn on **`PDU scanner - scanner mode`** (this pauses normal polling).
2. Set **`PDU scanner - from` / `to`** (0–65535) and press **`Scan range (from-to)`**, or use a preset button (`Scan PDU 0-999` … `Scan PDU 10000-19999`).
3. The Atom reads every address with function 0x03 (or 0x04 with the *Input function* switch) and publishes each answer as `OK PDU <N> = <value>` in `PDU scanner - last result` and in the ESPHome log.
4. Turn scanner mode off to resume normal polling.

Pasting such a dump into an issue — especially from a **multi-zone** install or a different firmware — is the most useful thing you can share. (The earlier "Zone 2/3 candidate" probes were removed: those addresses don't exist on 1- or 2-zone units.)

## Setup

1. `cp secrets.yaml.example secrets.yaml` and fill in your WiFi.
2. Flash `immergas-magis-pro-combo-tt-bms.yaml` with ESPHome 2026.9+ (or `...-beta.yaml` for the scanner).
3. Adopt in Home Assistant.

## Notes

- **Boiler fault descriptions (114 codes)** are the official Immergas English texts (extracted from the Dominus app labels).
- The Modbus register meanings are the author's reverse-engineering; corrections welcome.
- Renaming an entity in a new version creates a new entity in Home Assistant (the entity ID comes from the name); history stays with the old one.

## Disclaimer

Reverse-engineered for **interoperability / personal use**. Not affiliated with or endorsed by Immergas or Samsung. Writing to unknown/service registers can change appliance behaviour — use at your own risk.

## License

MIT — see [LICENSE](LICENSE).
