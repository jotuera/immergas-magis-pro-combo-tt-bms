# Changelog

## v2.3.0

- **Decoded 4201 = R01** (outdoor temperature probe for the weather curve): 0 = OFF, 1 = OU, 2 = IU. Found by @Bettapro in #4 and
  verified on a Magis Combo 9 Plus V2 (fw 6.0): R01 OFF switches D04 from the curve to a fixed flow setpoint.
- **New control:** select `R01 Outdoor temperature probe` in `tt_heating.yaml` (read every 30 s, writable).
  It replaces the diagnostic `PDU 4201`, which you can delete in HA.

## v2.2.0

- New substitution **`tt_name_prefix`** (default `"Immergas"`): the first part of every entity name
  (`"<prefix> - <name>"`). With the default the names are exactly as in v2.1.0. Set it in the main YAML when the
  T-/T+ packages share one device with dd-modbus / f1f2-nasa, e.g. `tt_name_prefix: "Immergas TT"`.

## v2.1.0

- **DHW boost:** switch `DHW boost` + `DHW boost duration` (5–60 min). Register 4011 = DHW tank temperature supplied by the BMS (×0.1 °C, 0 = own probe). The boost writes 4011 = 1, then 0. 4011 is cleared at every boot.
- **4011 read-out:** `PDU 4011` (diagnostics, 300 s) is replaced by `DHW temperature from BMS (4011)` (30 s). This is a new entity in HA.
- **Gas estimate:** DHW rate also when D22 = 1 with D98 215 / D99 8.

## v2.0.0

- **Modular config:**
  - The main YAML now only holds the device settings and a `packages:` list. The code lives in `packages/tt_*.yaml` and is fetched from GitHub.
  - The PDU scanner is the optional package `tt_scanner.yaml`, which replaces `immergas-magis-pro-combo-tt-bms-beta.yaml`.
  - File names and ids start with `tt_`, so the D+/D- and F1/F2 repos can be combined with this one on one device.
- **Languages:** `packages/lang/tt_en.yaml` and `packages/lang/tt_pl.yaml` (entity names, state texts, 114 fault descriptions).
- **New:** estimated gas consumption (`tt_gas.yaml`): m³ counter for the Energy dashboard, calibrated CH/DHW rates, "Gas boiler running".
- **Decoded:**
  - 4554 = D73 compressor discharge temperature
  - 4585 = D72 compressor temperature
  - 4586 = compressor power step (0-6)
  - 3027 = candidate "heat pump excluded"
- **Status codes:**
  - D97 336
  - D98 236, D98 215 = boiler heating (cycle)
  - D99 7, D99 44
  - D99 61 = DHW on the heat pump (was "solar boost")
- **Changed:** "Heating active" / "Cooling active" no longer use the `heat` / `cold` device classes (shown as on/off).
- **Renamed (new entities in HA):**
  - `Refrigerant hot-side temperature (cand.)` → `D73 Compressor discharge temperature`
  - `Refrigerant circuit 4585 (cand. temp D72-74)` → `D72 Compressor temperature`
  - `Compressor running` → `Compressor power step`

## v1.1.0

- Integration / installer settings as controls (I01–I09, T02, T04, P15, A12), D61 system model, D91 firmware, ESPHome 2026.9 cleanup.

## v1.0.0

- First release.
