# Immergas Magis Pro / Combo (Audax) — Modbus BMS via ESPHome (M5 Atom)

Read and control an **Immergas Magis Combo** (Audax heat pump) over the **T-/T+ (BMS) Modbus RS485** bus using an **M5Stack Atom + ESPHome**, integrated into Home Assistant.

This is a community reverse-engineering project — Immergas does not publish the Modbus register map for this bus, so the map below was recovered by scanning + cross-referencing the Magis Combo installer manual and the Samsung EHS/NASA documentation.

## Hardware

- **M5Stack Atom Lite** (ESP32) + isolated **RS485** transceiver (e.g. MAX3485 / M5 Isolated RS485 Unit)
- Wiring: Immergas **T+ / T-** → RS485 **A / B** (swap A/B if you get CRC errors)
- Bus: **Modbus RTU, slave 11, 9600 8N2**, 2-wire (the Immergas manual specifies 8N2; it also reads fine on 8N1 receivers, so either works)
- Tested on: **Magis Combo 9 Plus V2** (Audax 9, 9 kW) and **Magis Pro V2** (community-confirmed)
- The Atom is the Modbus **master**; the heat pump is slave 11.

## What it does

- **Reads**: flow/return/DHW/outdoor temperatures, EEV position, refrigerant-circuit temps, heat-pump/boiler/system status, fault code.
- **Controls**: operating mode (2000), DHW setpoint (2095), zone-1 heating/cooling limits (R04/R05/R12/R13), circulation pump min/max speed (A03/A04), zone-1 thermostat.
- **Decoded boiler fault** (114 Immergas anomaly codes), **calculated metrics** (Delta T, mean loop temp, thermal lift, hydraulic-separator delta).
- BMS connection toggle (start/stop polling).

## ⚠️ Important — operating-mode caveat

If a **Dominus / zone panel is active on the D+/D- bus**, it *owns* the operating mode: the boiler polls the panel and reverts any mode written via T-/T+ (register 2000). In that setup, change the mode from the panel/Dominus instead. When no panel is present, T-/T+ mode control works.

## Bus facts

- The unit answers **only Modbus function 0x03 (read holding registers)**. Functions 0x01 (coils), 0x02 (discrete inputs) and 0x04 (input registers) are rejected (illegal function).
- Register banks: **2xxx** (config/control), **3xxx** (sensors), **4xxx** (heat pump / refrigerant), **6xxx** (parameters / statuses).
- Immergas "PDU N" = holding register address N.

## Register map (known)

| Reg | Code | Description | Scale |
|---|---|---|---|
| 2000 | — | Operating mode (0=standby, 1=summer/DHW, 2=cooling, 3=winter) | RW |
| 2010 | — | Zone 1 thermostat / heating request (bit0) | RW |
| 2095 | D05 | DHW setpoint | ×0.1 °C, RW |
| 2100 | — | Fault code (0 = none, else E-code) | R |
| 3000 | D20 | System flow temperature | ×0.1 °C |
| 3001 | D08 | Heat pump return water temperature | ×0.1 °C |
| 3002 | D06 | Outdoor temperature | ×0.1 °C |
| 3016 | D03 | Storage tank unit temperature | ×0.1 °C |
| 3029 | D23 | Indoor Unit return temperature | ×0.1 °C |
| 3054 | D14 | Circulator pump flow rate | l/h |
| 3056 | D24 | Chiller circuit liquid temperature | ×0.1 °C |
| 4202 | A11 | Outdoor Unit model | code (9 = 9 kW) |
| 4350 | R05 | Zone 1 minimum central heating | °C, RW |
| 4351 | R04 | Zone 1 maximum central heating | °C, RW |
| 4356 | R13 | Zone 1 maximum cooling | °C, RW |
| 4357 | R12 | Zone 1 minimum cooling | °C, RW |
| 4557 | D77 | Electronic expansion valve (EEV) position | 0–2000 |
| 6000 | T05 | Central heating ignitions timer | RW |
| 6010 | A03 | Circulation pump minimum speed | %, RW |
| 6011 | A04 | Circulation pump maximum speed | %, RW |
| 6500 | D97? | Heat pump demand status (candidate) | 0–999 |
| 6501 | D98? | Thermal generator demand status (candidate) | 0–999 |
| 6502 | D99 | System state (0/82=Standby, 6=Heating, 8=Heating cycle, 41=Off, **62=DHW heating**) | 0–999 |
| 6506/6507 | D140/D141 | Internal RTC hour / minute (read-only, free-running) | h / min |

## 🔎 Unknown registers — contributions welcome!

Many registers respond but their meaning is **not yet confirmed** — exposed as `PDU <N>`, `Register <N> (RW 0-1)`, or `Refrigerant circuit <N> (cand.)`. They come alive during compressor operation and look like circuit temperatures / states.

If you own an Immergas Magis Combo / Audax and can correlate these values with operating states, **please open an issue or PR** — that's how we finish the map.

## 🧪 Beta config — for multi-zone testers

`immergas-magis-pro-combo-tt-bms-beta.yaml` is a drop-in copy of the stable config **plus an experimental "Zone 2/3 candidates" group**: read-only probes on registers `3004`, `3005` and `4352`–`4355`. These are map gaps that stay **unavailable on a single-zone system** (expected) and are suspected to come alive on **2- or 3-zone** installs — the addresses sit right next to the confirmed Zone 2 register `R08` (3003) and in the gap between Zone 1 heating (4350/4351) and cooling (4356/4357).

If you run a multi-zone Magis Combo / Pro V2, flashing the beta and reporting which of these show plausible °C values (and whether they track your Zone 2/3 setpoints) would directly help finish the map. **Issues/PRs very welcome.** The stable config is unaffected.

The beta also bundles an **active PDU scanner**. Turn on the `PDU scanner - scanner mode` switch (this pauses normal polling), then press a `scan registers N-N` button. The Atom sweeps that range with Modbus function 0x03 (or 0x04 if you flip the *Input function* switch) and logs every response as `OK reg <N> = <value>` to the ESPHome log — a full dump of which registers answer on your system. Turn the switch back off to resume normal polling. Pasting that log into an issue is the single most useful thing a multi-zone owner can share.

## Setup

1. `cp secrets.yaml.example secrets.yaml` and fill in your WiFi.
2. Flash `immergas-magis-pro-combo-tt-bms.yaml` with ESPHome (or `...-beta.yaml` if you're helping test multi-zone).
3. Adopt in Home Assistant.

## Notes / TODO

- **Boiler fault descriptions (114 codes)** are the official Immergas English texts (extracted from the Dominus app labels).
- The Modbus register meanings are the author's reverse-engineering; corrections welcome.

## Disclaimer

Reverse-engineered for **interoperability / personal use**. Not affiliated with or endorsed by Immergas or Samsung. Writing to unknown/service registers can change appliance behaviour — use at your own risk.

## License

MIT — see [LICENSE](LICENSE).
