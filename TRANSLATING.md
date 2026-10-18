# Translating

Entity names and the texts shown in Home Assistant come from one language file in `packages/lang/`:

| File | Language | Status |
|---|---|---|
| `tt_en.yaml` | English | complete (default) |
| `tt_pl.yaml` | Polski | complete |

## How it works

Each file is a block of `substitutions:`:

- `n_tt_<id>`: name of the entity with that `id`. Keep the `Immergas - ` prefix and the manual codes such as `D71`, `I05` or `R04`.
- `t_tt_*`: texts shown as states or select options (D97/D98/D99 codes, operating modes, I05 modes, "No faults", …).
- `l_tt_fault_cases`: C++ `case` lines with the 114 boiler fault descriptions (used in the "Boiler fault (description)" lambda). Keep the `case N: t = "..."; break;` form and don't use `$` in the texts.

## Adding a language

1. Copy `packages/lang/tt_en.yaml` to `packages/lang/tt_<code>.yaml` and translate the values (not the keys).
2. Keep select texts unique inside each select.
3. Validate with `esphome config` using your file in the `packages:` list, then open a pull request.

Note: the entity name is part of the entity identity in Home Assistant. After a language change, HA creates new entities.
