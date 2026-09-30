# ldo-loadreg-setup

Two SKILL files. `ldo_loadreg_setup.il` adds an **LDO load-regulation test** to an ADE Assembler
setup you already have: the DC sweep, the six output expressions and the two pass/fail specs, in one
`load()`. `ldo_pvt_corners.il` adds the **shared PVT corner set** (5 processes × VIN ±10% ×
−40/25/125 °C = 45 points), enables it for every test, and works on ADE XL and ADE Assembler views.

**Step 1 — grab both files** into the same directory:

```sh
curl -O https://raw.githubusercontent.com/borenw/ldo-loadreg-setup/main/ldo_loadreg_setup.il
curl -O https://raw.githubusercontent.com/borenw/ldo-loadreg-setup/main/ldo_pvt_corners.il
```

**Step 2 — edit sections 1 to 3** at the top of the file. Nothing else in it needs touching:

| Section | What you set |
|---|---|
| 1. Where the setup lives | `LIB`, `CELL`, `VIEW` (`adexl` or `maestro`), `TB_VIEW`, `TEST` |
| 2. Net and instance names | `DUT`, `DUT_POWER`, `DUT_GND`, `DUT_OUTPUT`, `ILOAD_VAR`, `VIN_VAR` |
| 3. Operating point and specs | `VIN_VAL`, `VOUT_NOM`, `ILOAD_MIN/MAX`, `NPTS`, `SPEC_LOADREG`, `SPEC_ERR`, `PVT`, `TEMPS`, `RUN` |

**Step 3 — close the setup view in the GUI** (or open it read-only). The script opens it for
editing and will error out with a clear message if Virtuoso still holds it.

**Step 4 — run it from the CIW:**

```lisp
load("ldo_loadreg_setup.il")
```

With `RUN t` (the default) it sets up the test, saves, simulates, waits, and drops
`./LDO_LoadReg_results.csv` beside your working directory. Set `RUN nil` to only build the test
and leave the running to you.

## Shared PVT corners

| Dimension | Values | How it is set |
|---|---|---|
| Process | TT, SS, FF, SF, FS | model sections swapped per corner: `tt`→`ss`/`ff`/`sf`/`fs` and `tt_3v`→`*_3v`; `tt_res`/`tt_mim` follow SS and FF only |
| Voltage | `VIN_VAL` −10%, typ, +10% | corner variable `vin` |
| Temperature | −40, 25, 125 °C | corner variable `temperature` |

The nominal model list comes from the setup's first test, and every section not in the swap
table stays nominal. Edit `ldoPvtProcess` at the top of `ldo_pvt_corners.il` if your PDK's
section names differ.

With `PVT t`, `ldo_loadreg_setup.il` adds the set itself. For a setup that already has its tests,
such as the other 14 LDR checks, edit section 1 of `ldo_pvt_corners.il` and run
`load("ldo_pvt_corners.il")` on its own. Checks that sweep VIN (dropout, line regulation) or
temperature (thermal shutdown) should get a P T or P V set:

```lisp
ldoPvtCorners(sdb models ?vary '(P T) ?tests '("LDO_Dropout") ?suffix "_PT")
```

## LDO LDR checklist page

`index.html` is the full low-level design review page this script comes from. It holds all 15 simulation checks (#1 Stability to #15 EM / IR and aging), and for each one it shows the testbench with the stimulus and measured nets, PVT or Monte Carlo charts with the spec lines, a worst-case table, and the ADE XL output expressions. The page also carries the load regulation worked example, a copy of this script, and a reference list of public app notes and papers.

Open it straight in a browser, or turn on GitHub Pages for `main` (root) to host it at
`https://borenw.github.io/ldo-loadreg-setup/`.

The chart data in the page is illustrative, shaped like a 1.2 V, 300 mA LDO. Replace it with your own simulation CSV.

## What your testbench must already have

The script edits the **setup**, not the schematic. Before running it, `tb_ldo/schematic` needs:

- a **DC current sink** from `DUT_OUTPUT` to ground, value = `ILOAD_VAR`
- a **DC voltage source** on `DUT_POWER`, value = `VIN_VAR`
- the **output cap and ESR** already placed

## What it measures

A DC sweep of `iload` from `ILOAD_MIN` to `ILOAD_MAX` (`NPTS` linear points, op-point saved),
then:

| Output | Expression | Spec |
|---|---|---|
| `VOUT` | `VS("/VOUT")` over the sweep | — |
| `VOUT_light` | `VOUT` at `ILOAD_MIN` | — |
| `VOUT_heavy` | `VOUT` at `ILOAD_MAX` | — |
| `load_reg_mV_per_A` | `1000·ΔVOUT / Δiload` | `< SPEC_LOADREG` (20 mV/A) |
| `VOUT_err_pct` | `100·max|VOUT − VOUT_NOM| / VOUT_NOM` | `< SPEC_ERR` (1.5 %) |
| `headroom_V` | `VIN − VOUT` at `ILOAD_MAX` | — |

Defaults are a 1.8 V → 1.2 V, 300 mA rail. Open the view after the run to see pass/fail per
corner.

## Requirements and caveats

- Needs the `mae*` SKILL API: **IC6.1.8 with ADE Assembler**, or **IC23 / IC25**.
- `mae*` cannot open a classic ADE XL view (`data.sdb`); `maeOpenSetup` returns nil with
  `ASSEMBLER-8036`. Build the test in a maestro view, then convert it with
  [maestro-to-adexl-gui](https://github.com/borenw/maestro-to-adexl-gui) if you need ADE XL.
  `ldo_pvt_corners.il` uses the `axl*` API and opens either view type.
- The DC sweep needs `sweep "Design Variable"` and `incrType "Linear"`. Without them ADE stores the
  keys but netlists a bare operating point, so check the printed DC analysis line. `lin` is a step
  count, so `NPTS 31` gives 32 points.
- Output expressions read design variables as `VAR("VOUT_NOM")`. A bare `VOUT_NOM` is unbound.
- `maeSetAnalysis` changed argument order between releases, so the script tries both forms. If
  neither takes, it prints the sweep you should enter by hand rather than failing silently — and
  it echoes back the DC analysis it actually ended up with. **Check that printed line.**
- It refuses to run if a test named `TEST` already exists, so it will not clobber your work.
- Checked on IC6.1.8 against a 1.8 V capless LDO testbench. Both scripts built the setups in batch
  mode on ADE XL and Assembler views, the nominal test ran in the ADE XL GUI, and all 45 corner
  points were checked by running Spectre on the netlist ADE generated. A batch `maeRunSimulation` built the netlist but never started a
  job on that host, so run from the GUI if `RUN t` stalls.
