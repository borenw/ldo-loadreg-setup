# ldo-loadreg-setup

One SKILL file that adds an **LDO load-regulation test** to an ADE XL / ADE Assembler setup you
already have — the DC sweep, the six output expressions and the two pass/fail specs, in one
`load()`.

**Step 1 — grab the file.** On this repo's page click **`ldo_loadreg_setup.il`** → **Download** (⤓),
or from a shell:

```sh
curl -O https://raw.githubusercontent.com/borenw/ldo-loadreg-setup/main/ldo_loadreg_setup.il
```

**Step 2 — edit sections 1 to 3** at the top of the file. Nothing else in it needs touching:

| Section | What you set |
|---|---|
| 1. Where the setup lives | `LIB`, `CELL`, `VIEW` (`adexl` or `maestro`), `TB_VIEW`, `TEST` |
| 2. Net and instance names | `DUT`, `DUT_POWER`, `DUT_GND`, `DUT_OUTPUT`, `ILOAD_VAR`, `VIN_VAR` |
| 3. Operating point and specs | `VIN_VAL`, `VOUT_NOM`, `ILOAD_MIN/MAX`, `NPTS`, `SPEC_LOADREG`, `SPEC_ERR`, `RUN` |

**Step 3 — close the setup view in the GUI** (or open it read-only). The script opens it for
editing and will error out with a clear message if Virtuoso still holds it.

**Step 4 — run it from the CIW:**

```lisp
load("ldo_loadreg_setup.il")
```

With `RUN t` (the default) it sets up the test, saves, simulates, waits, and drops
`./LDO_LoadReg_results.csv` beside your working directory. Set `RUN nil` to only build the test
and leave the running to you.

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
- `maeSetAnalysis` changed argument order between releases, so the script tries both forms. If
  neither takes, it prints the sweep you should enter by hand rather than failing silently — and
  it echoes back the DC analysis it actually ended up with. **Check that printed line.**
- It refuses to run if a test named `TEST` already exists, so it will not clobber your work.
- Not validated against every install. Verify the DC analysis settings on first use.
