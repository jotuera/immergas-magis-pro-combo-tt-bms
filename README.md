# Immergas Magis Pro / Combo (Audax) — Modbus BMS via ESPHome (M5 Atom)

🇬🇧 English | 🇵🇱 [Polski](README.pl.md)

<p align="center">
  <img src="https://img.shields.io/github/v/tag/jotuera/immergas-magis-pro-combo-tt-bms?label=version&color=blue" alt="Version">
  <img src="https://img.shields.io/badge/ESPHome-2026.9%2B-41BDF5.svg" alt="ESPHome 2026.9+">
  <img src="https://img.shields.io/badge/platform-ESP32-41BDF5.svg" alt="ESP32">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-green.svg" alt="License: MIT"></a>
</p>

Read and control an **Immergas Magis Pro / Combo** (Audax heat pump) over the **T-/T+ (BMS) Modbus RS485** bus using an **M5Stack Atom + ESPHome**, integrated into Home Assistant.

This is a community reverse-engineering project. Immergas does not publish the Modbus register map for this bus, so the map below was recovered by scanning, cross-referencing the Magis Combo installer manual, live logging of operating cycles and comparing with the Samsung NASA (F1/F2) bus.

Sister projects (same author, designed to run together on one device later):
- [immergas-magis-pro-combo-dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus) — Dominus / zone-panel emulator on the D+/D- bus
- [immergas-magis-pro-combo-f1f2-nasa](https://github.com/jotuera/immergas-magis-pro-combo-f1f2-nasa) — Samsung NASA reader on the F1/F2 bus

> [!IMPORTANT]
> **A zone 1 panel on D+/D- is required for heating.** While this config polls the T-/T+ bus, the unit takes the BMS as its controller. Without a zone panel on D+/D-, the **heat pump stops and does not start again**, even if your room thermostat (40-1/41) calls for heat. It runs again as soon as polling stops (`BMS connection` off).
>
> Connect one of these on **D+/D-**:
> - a physical **Immergas zone 1 remote panel** (code **3.030863**), or
> - the **zone 1 panel emulator** from [immergas-magis-pro-combo-dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus) (second ESP on D+/D-).
>
> This happens even with only a few read-only sensors enabled: the polling itself triggers it, not any write. Register 2010 does not replace the panel on firmware ≤ 6.0. See [#1](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/1) and [#3](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/3).

## Hardware

- **M5Stack Atom Lite** (ESP32) + **RS485** transceiver (e.g. MAX3485 / M5 RS485 unit)
- Wiring: Immergas **T+ / T-** → RS485 **A / B** (swap A/B if you get CRC errors)
- Bus: **Modbus RTU, slave 11, 9600 8N2**, 2-wire (the Immergas manual specifies 8N2; 8N1 receivers also work)
- Tested on: **Magis Combo 9 Plus V2** (Audax 9, 9 kW, firmware D91 = 6.0) and **Magis Pro V2** (community-confirmed)
- The Atom is the Modbus **master**; the heat pump is slave 11.

## What's new in v2.1.0

- **DHW boost** (`tt_dhw.yaml`): the switch `DHW boost` makes the unit heat domestic hot water now, for the time set in `DHW boost duration` (5–60 min, default 20). It uses register **4011 = DHW tank temperature supplied by the BMS** (×0.1 °C, 0 = the unit's own probe). The boost writes 4011 = 1 (0.1 °C, "tank cold") and writes 0 when it ends.
  - **Safety:** 4011 is always cleared at boot, so a reboot or power cut never leaves an override behind.
  - **Probe hidden:** during the boost D03 and the DHW entity show 0.1 °C instead of the real tank temperature.
  - Found by @Bettapro ([issue #1](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/1)) and verified on a Magis Combo 9 Plus V2: 4011 = 1 → D03 0.1 °C, gas DHW starts; 4011 = 5 → D03 0.5 °C; 4011 = 0 → real 55 °C, DHW stops.
- `PDU 4011` is replaced by `DHW temperature from BMS (4011)`, read every 30 s (new entity in HA).
- **Gas estimate fix:** DHW on gas is also recognised when the generator 3-way valve D22 is on DHW and the unit reports D98 215 / D99 8. Both the start of a DHW cycle and the boost report it this way instead of 62.

## What's new in v2.0.0

- **Modular config.** The single 1300-line YAML is split into packages (`packages/tt_*.yaml`) that ESPHome downloads from GitHub. You copy only the short main file and choose which features you want. The separate `-beta.yaml` is gone; the PDU scanner is now an optional package (`tt_scanner.yaml`).
- **Languages:** entity names and texts come from one language file: `packages/lang/tt_en.yaml` (English) or `packages/lang/tt_pl.yaml` (Polski). Any single name can be overridden in `substitutions:`.
- **Estimated gas consumption** (`tt_gas.yaml`): boiler run time × calibrated m³/h for CH and DHW, as a `total_increasing` m³ counter for the Home Assistant Energy dashboard.
- **Newly decoded:** 4554 = **D73 compressor discharge temperature**, 4585 = **D72 compressor temperature** (both match the Samsung NASA values to 0.1 °C), 4586 = **compressor power step 0-6** (not an on/off flag).
- **New status codes:** D99 7 / D97 336 / D98 236 (heating by the boiler only, heat pump excluded by I10), D99 44 (end of DHW on the heat pump), D99 61 = **DHW on the heat pump** (not "solar boost"), D98 215 = boiler heating (cycle).
- "Heating active" / "Cooling active" no longer use the `heat` / `cold` device classes, so they show on/off like "DHW active".
- File names and ids start with `tt_` so this repo never clashes with the D+/D- and F1/F2 projects.

⚠️ **Upgrading from 1.x:**
- **Rename:** three read-outs were renamed and show up as **new entities**: `Refrigerant hot-side temperature (cand.)` → `D73 Compressor discharge temperature`, `Refrigerant circuit 4585 (cand. temp D72-74)` → `D72 Compressor temperature`, `Compressor running` → `Compressor power step`. Delete the old ones in HA.
- **Unchanged:** all other entity names are the same, so their history continues.
- **Switching:** replace your YAML with the new main file. Copy your Wi-Fi / static IP settings into it and keep the same `device_name`.

## Project structure

```
immergas-magis-pro-combo-tt-bms.yaml   main file: device, Wi-Fi, API + list of packages (the only file you copy)
packages/
  tt_core.yaml          UART, Modbus, the two polling controllers, "BMS connection" switch      (required)
  tt_status.yaml        operating mode, D97/D98/D99, boiler fault code + text, A11, D61, D91,    (required)
                        internal clock, "Heating/Cooling/DHW active"
  tt_circuit.yaml       D20/D08 flow/return, boiler flow (3029), D14, D06, D04, D22 3-way valve  (required)
  tt_heat_pump.yaml     D24, D71 frequency, D72/D73/D74, D76 fan, D77 EEV, compressor power step
  tt_dhw.yaml           D03 tank temperature, DHW water_heater (2095), T02, DHW boost (4011)
  tt_heating.yaml       outdoor probe R01, zone 1 limits R04/R05/R12/R13, zone 1 thermostat enable (2010)
  tt_integration.yaml   menu I: I01, I02, I03, I04, I05, I07, I09, I10
  tt_installer.yaml     P15, A12, A03/A04 pump speed, T04, T05, T07
  tt_computed.yaml      Delta T, thermal power, mean loop temperature, thermal lift, HP return - boiler flow
  tt_gas.yaml           estimated gas consumption (needs tt_status + tt_circuit)
  tt_diagnostics.yaml   undecoded registers + undocumented writable 0/1 registers
  tt_scanner.yaml       optional active PDU scanner
  lang/tt_en.yaml       English names/texts
  lang/tt_pl.yaml       Polish names/texts
```

To drop a feature, delete its line from `packages:` in the main file. `tt_core`, `tt_status` and `tt_circuit` are always needed.

## Setup

1. Copy `immergas-magis-pro-combo-tt-bms.yaml` into ESPHome (dashboard → new device → paste) and `secrets.yaml.example` → `secrets.yaml` (Wi-Fi, fallback AP password).
2. Pick the language line (`packages/lang/tt_en.yaml` or `tt_pl.yaml`). If your RS485 pins differ, set `tt_uart_tx_pin` / `tt_uart_rx_pin` / `tt_uart_de_pin` in `substitutions:`.
3. Install with **ESPHome 2026.9 or newer**, then adopt the device in Home Assistant.
4. To upgrade later, change `ref: v2.0.0` to a newer tag (or `main` for the development version).

## What it does

- **Reads:** flow/return/DHW/outdoor temperatures, **calculated flow setpoint (D04)**, compressor frequency and power step, compressor/discharge/evaporator temperatures, EEV position, outdoor fan speed, 3-way valve position, heat-pump/boiler/system status, fault code + description, system model (D61), firmware version (D91).
- **Controls:** operating mode (2000), DHW setpoint (2095), **DHW boost (4011)**, weather-curve outdoor probe (R01), zone-1 heating/cooling limits (R04/R05/R12/R13), circulation pump min/max speed (A03/A04), zone-1 thermostat, **gas/heat-pump integration (I01–I10)**, T02, T04, T05, T07, P15 anti-legionella, A12 system vent.
- **Calculated:** Delta T, mean loop temperature, thermal lift, heat-pump return − boiler flow difference, **estimated gas consumption**.
- BMS connection toggle (start/stop polling).

## Estimated gas consumption

The unit does not report gas use, so `tt_gas.yaml` counts the time the gas boiler runs and multiplies it by an average m³/h, with separate rates for CH and DHW.
- **When it counts:** boiler running = D99 62 (DHW on gas), D98 215/236 or D99 7 (CH by the boiler). In heating (D99 6/8) it also counts while the boiler flow is at least 8 K above the heat-pump return. Normal heat-pump heating (D98 252) is not counted.
- **Calibration:** read your gas meter at the start and end of a week. Divide the real m³ by the increase of `Gas estimated consumption`, then multiply both rates (`Gas consumption CH/DHW (calibration)`) by that factor.
- **Energy dashboard:** add `Gas estimated consumption` under Settings → Dashboards → Energy → Gas consumption. If the counter ever jumps or is reset, check the gas cost statistic (Developer tools → Statistics) for a spike.

## ⚠️ Important caveats

- **Operating mode: whoever keeps asserting it wins.** An *emulated* zone panel on D+/D- re-asserts its mode continuously and reverts any mode written via T-/T+ (register 2000). With a *real* Dominus panel (firmware 6.0), the unit holds a mode written over T-/T+ as the BMS command. It then reverts mode changes made on the keypad or on the Dominus. With a T-/T+ master running, change the mode from Home Assistant.
- **Heating needs a zone 1 panel on D+/D- (see the note at the top).** With the BMS master polling, the heat pump runs **only** when a zone 1 panel is present on D+/D-: the real Immergas zone panel (3.030863; A31 = RPT or RP) or the [dd-modbus emulator](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus).
  - **Without a panel:** the unit stops heating as soon as polling starts and ignores the 40-1/41 thermostat contact. Register 2010 doesn't help. Users with fw 4.3 and 6.0 have confirmed this (#3: firmware not reported).
  - **Firmware 8.3+:** reports that 2010 works come only from these units.
  - See [#1](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/1) and [#3](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/3).
- **Probes not fitted read `-32768` (0x8000),** e.g. register 3029 (boiler flow) on a heat-pump-only Magis Pro. All temperature sensors map this to *unknown*.

## Bus facts

- **Function codes:** the unit answers **only Modbus function 0x03**. Functions 0x01, 0x02 and 0x04 are rejected (illegal function). Writes use 0x06.
- **Register banks:** **2xxx** (config/control), **3xxx** (sensors), **4xxx** (heat pump / refrigerant), **6xxx** (parameters / statuses). The whole range **0–19999** has been scanned; nothing answers above 6510.
- **No Victrix / Magis M TOP interface:** the low BMS registers from the Victrix and Magis M TOP manuals are not available on the Magis V2 (fw 6.0). 512 is **read-only** (a write returns exception 2), 256 and the 0x04 input registers don't exist, and 61440–61442 (Modbus config) return exception 2. There is no BMS heat request and no BMS "silent mode" on this bus.
- **Fixed map:** entering the installer PIN does **not** expose more registers. Heating-curve breakpoints (R02/R03) and most menu parameters are not on this bus.
- **Menu banks:** the **44xx** bank holds integration / installer settings (menus I, T, P). The **65xx** bank holds info read-outs (menu D: D97–D99, D61, D91, internal clock).
- **Settings not exposed on T-/T+** (changed on the unit, no register moved): I08, P07 (but D06 reads *after* the P07 correction), P11, P13, P21, P23, P24, T09, T23, **U11 night function** (it goes to the outdoor unit as a 30 % frequency limit on D+/D-, not here), A05, A18, A31, A41, maintenance menu M (M13, M15–M17). Zone remote-panel settings live on the D+/D- bus.
- **Maintenance menu:** entering menu **M** sets fault code **193** (E193 "test mode") in register 2100 until you leave it.
- **Naming:** Immergas "PDU N" = holding register address N.

## Register map (known)

| Reg | Code | Description | Scale |
|---|---|---|---|
| 512 | — | Read-only on the Magis V2 (always 0 since Oct 2026; 280/2000 seen once in June) | — |
| 2000 | — | Operating mode (0=standby, 1=summer/DHW, 2=cooling, 3=winter) | RW |
| 2010 | — | Zone 1 thermostat / heating request (bit0) — see caveat above | RW |
| 2095 | D05 | DHW setpoint | ×0.1 °C, RW |
| 2100 | — | Fault code (0 = none, else E-code) | R |
| 3000 | D20 | System flow temperature | ×0.1 °C |
| 3001 | D08 | Heat pump return water temperature | ×0.1 °C |
| 3002 | D06 | Outdoor temperature | ×0.1 °C |
| 3003 | D04 | **Calculated flow setpoint** from the heating curve; 0 = no heat demand | ×0.1 °C |
| 3016 | D03 | Storage tank unit temperature | ×0.1 °C |
| 3027 | — | **Heat pump excluded**: 1 while only the boiler heats (outdoor temperature below I10, D99 7) or during a heat pump communication loss (E189) | 0/1 |
| 3028 | D22 | Generator 3-way valve (1 = DHW, 0 = CH) | 0/1 |
| 3029 | — | **Secondary generator (gas boiler) flow temperature**; `-32768` on units without a boiler | ×0.1 °C |
| 3054 | D14 | Circulator pump flow rate (0 while the pump is stopped) | l/h |
| 3056 | D24 | Chiller circuit liquid temperature | ×0.1 °C |
| 4011 | — | **DHW tank temperature supplied by the BMS**: overrides the tank probe while > 0 (D03 shows this value, the unit heats DHW as if the tank were cold); 0 = own probe | ×0.1 °C, RW |
| 4201 | R01 | Outdoor temperature probe for the weather curve: 0 = OFF (fixed flow setpoint), 1 = OU (outdoor unit probe), 2 = IU (probe on the indoor unit) | enum, RW |
| 4202 | A11 | Outdoor Unit model | code (9 = 9 kW) |
| 4350 | R05 | Zone 1 minimum central heating | °C, RW |
| 4351 | R04 | Zone 1 maximum central heating | °C, RW |
| 4356 | R13 | Zone 1 maximum cooling | °C, RW |
| 4357 | R12 | Zone 1 minimum cooling | °C, RW |
| 4402 | T02 | DHW thermostat (DHW switch-on differential) | °C, RW |
| 4403 | P15 | Anti-legionella function | 0/1, RW |
| 4411 | T04 | (not described in any Immergas manual) | RW |
| 4451 | I05 | Integration activation mode (0 = AU automatic, 1 = MA manual) | RW |
| 4453 | I02 | System (CH) integration enabling (1 = AL, 0 = OFF) | 0/1, RW |
| 4454 | I01 | DHW integration enabling (1 = AL, 0 = OFF) | 0/1, RW |
| 4449 | — | **Force boiler heating**: 1 = heat pump stopped, the boiler heats (D99 7, D97 351, D98 251) | 0/1, RW |
| 4452 | I10 | Central heating (system) integration activation temperature | °C as a signed byte in the low byte (251 = −5 °C), RW |
| 4455 | I09 | DHW integration activation temperature | °C as a signed byte in the low byte (251 = −5 °C), RW |
| 4468 | I04 | Central heating max wait time | min, RW |
| 4469 | I03 | DHW max wait time | min, RW |
| 4479 | I07 | Activation band | °C, RW |
| 4551 | D74 | **Evaporator (outdoor coil) temperature** (= NASA 0x8218) | ×0.1 °C, signed |
| 4554 | D73 | **Compressor discharge temperature** (= NASA 0x820A) | ×0.1 °C |
| 4557 | D77 | Electronic expansion valve (EEV) position (2000 = fully open at rest) | 0–2000 |
| 4558 | D71 | **Compressor frequency** | Hz |
| 4585 | D72 | **Compressor temperature** (top, = NASA 0x8280) | ×0.1 °C |
| 4586 | — | **Compressor power step** 0–6 (25 Hz = 1, 30 = 2, 43 = 3, 51 = 4, 58 = 5, 61 = 6) | step |
| 4587 | D76 | Outdoor unit fan speed (candidate) | rpm |
| 4600 | — | **Start system venting** now (E139, D99 42, panel "AP99"); 0 stops it | 0/1, RW |
| 6000 | T05 | Central heating ignitions timer | RW |
| 6002 | T07 | Room thermostat (TA) request delay: 0-24 = 0-240 s in 10 s steps (panel value) | ×10 s, RW |
| 6010 | A03 | Circulation pump minimum speed | %, RW |
| 6011 | A04 | Circulation pump maximum speed | %, RW |
| 6013 | A12 | System vent (automatic venting at power-up) | 0/1, RW |
| 6500 | D97 | Heat pump demand status — see below | 0–999 |
| 6501 | D98 | Thermal generator demand status — see below | 0–999 |
| 6502 | D99 | System state — see below | 0–999 |
| 6505 | D91 | P.C.B. software version (60 = 6.0, 83 = 8.3) | ×0.1 |
| 6506/6507 | D140/D141 | Internal RTC hour / minute (read-only) | h / min |
| 6510 | D61 | System model: 1 = MP, 2 = MCI, 3 = MCP, 4 = MPH, 5 = MCH, 6 = MCPH | code |

All RW settings in the 44xx / 60xx rows were verified both ways: changing them on the keypad moves the register, and writing them from Home Assistant changes the value on the unit.

**D97 heat pump demand:** 300 standby · 332/390 heating · 352 heating (thermostat call) · 322 DHW on the heat pump (fw 4.3) · 342 cooling request · 336 excluded by the I10 threshold (boiler heats) · 351 stopped by the BMS (4449, boiler heats).
**D98 thermal generator demand:** 200 standby · 215 boiler heating (cycle) · 232 heating · 252 heating (thermostat call, boiler not firing) · 236 heating by the boiler only · 251 boiler heating forced by the BMS (4449).
**D99 system state:** 0 / 82 standby · 1 cooling request · 6 heating request · 7 heating by the boiler only (heat pump excluded by I10) · 8 heating cycle · 41 pump overrun after CH (~3 min) · 42 system venting (4600) · 44 end of DHW on the heat pump (~3 min) · 45 end of DHW (~1 min) · 61 DHW on the heat pump · 62 DHW on the gas boiler (Combo) · 64 DHW with the electric heater. Codes were observed on fw 6.0. On fw 4.3, plain DHW was reported as 61.

## 🔎 Unknown registers — contributions welcome!

Still unidentified:
- **Writable 0/1:** `2102` (0; a written 1 falls back to 0 within seconds with no visible effect).
- **Always zero:** `512, 3061, 4588, 6014, 6503, 6504` (512, 3061 and 6504 moved once in June, possibly in cooling).

If you can correlate any of these with operating states, **please open an issue or PR**.

## 🧪 PDU scanner (optional package)

Uncomment `packages/tt_scanner.yaml` in the main file:

1. Turn on **`PDU scanner - scanner mode`** (this pauses normal polling).
2. Set **`PDU scanner - from` / `to`** (0–65535) and press **`Scan range (from-to)`**, or use a preset button.
3. Every address is read with 0x03 (or 0x04 with the *Input function* switch). Each answer appears as `OK PDU <N> = <value>` in `PDU scanner - last result` and in the ESPHome log.
4. Turn scanner mode off to resume normal polling.

A dump from a **multi-zone** install or a different firmware is the most useful thing you can share.

## Running on one device with the other buses

The `tt_` packages can share one ESP32 with [dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus) and [f1f2-nasa](https://github.com/jotuera/immergas-magis-pro-combo-f1f2-nasa): put the package lists next to each other under one `packages:` and keep one device section (esp32, wifi, api, logger). Each bus needs its own UART and pins (`tt_uart_tx_pin` / `tt_uart_rx_pin` / `tt_uart_de_pin`); an ESP32 has three hardware UARTs, so move the logger off them (`logger: baud_rate: 0`, or `hardware_uart: USB_SERIAL_JTAG` on an ESP32-S3). To tell the buses apart in Home Assistant, set a name prefix per bus, e.g. `tt_name_prefix: "Immergas TT"` → *Immergas TT - D20 System flow temperature*.

## Notes

- **Fault descriptions:** the 114 boiler fault descriptions are the official Immergas texts (English and Polish, from the Dominus app labels).
- **Register meanings:** the Modbus register meanings are the author's reverse-engineering; corrections welcome.
- **Renames create new entities:** renaming an entity creates a new entity in Home Assistant (the entity ID comes from the name), and history stays with the old one. The same happens when you switch the language file.
- **Translations:** see [TRANSLATING.md](TRANSLATING.md).

## Disclaimer

Reverse-engineered for **interoperability / personal use**. Not affiliated with or endorsed by Immergas or Samsung. Writing to unknown/service registers can change appliance behaviour — use at your own risk.

## License

MIT — see [LICENSE](LICENSE).
