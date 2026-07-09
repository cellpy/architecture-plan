# Legacy ⇄ cellpy-core header swapping — how it works today

**Date:** 2026-07-08
**Companion diagram:** [legacy-cellpy-core-header-swapping.excalidraw](legacy-cellpy-core-header-swapping.excalidraw)

This document describes how column-header translation between **legacy cellpy**
(pandas frames named by `HeadersNormal` / `HeadersStepTable` / `HeadersSummary`) and
the **cellpy-core native schema** (polars frames named by `RawCols` / `StepCols` /
`CycleCols`) is implemented today.

> **Not to be confused with:** `cellpy/parameters/legacy/update_headers.py`, which
> translates *old cellpy-file versions* (v4–v7 header names) to the current legacy
> names when loading old `.h5` files. That is a separate, one-way migration and is
> only mentioned here for disambiguation.

---

## 1. The three worlds

| | cellpy (legacy) | the bridge | cellpy-core (native) |
|---|---|---|---|
| **Code** | `cellpy/readers/cellreader.py` | `cellpycore/cell_core.py` → `OldCellpyCellCore` (line 719) | `cellpycore/config.py`, `cellpycore/summarizers.py` |
| **DataFrames** | pandas | pandas ⇄ polars seam | polars |
| **Header definitions** | `parameters/internal_settings.py`: `HeadersNormal`, `HeadersStepTable`, `HeadersSummary` | verbatim mirrors in `cellpycore/legacy/headers.py` | `RawCols`, `StepCols`, `CycleCols`, bundled in a frozen `Schema(raw, cycle, step)` |
| **Example names** | `voltage`, `cycle_index`, `data_point`, `charge`, `rate_avr` | — | `potential`, `cycle_num`, `datapoint_num`, `charge_capacity_mean`, `c_rate` |

The native engines are **schema-agnostic**: they read every column name from the
injected `Schema` object, never from module globals. Units are handled by value —
cellpy computes conversion factors with pint and passes plain floats across the seam,
so cellpy-core has no pint dependency.

## 2. The single source of truth: `cellpycore/legacy/mapping.py`

All translation is declared **once**, in `cellpycore/legacy/mapping.py`, as pairs of
**column-name strings** (the dataclass *values*, not the attribute names — that is
what `DataFrame.rename` acts on, and it sidesteps legacy `HeadersSummary` having two
attributes with the same value: `discharge_capacity` / `discharge_capacity_raw`).

### 2.1 The pair tables

| Table | Size | Examples (native → legacy) |
|---|---|---|
| `RAW_PAIRS` | 10 | `potential → voltage`, `cycle_num → cycle_index`, `step_num → step_index`, `datapoint_num → data_point`, `cumulative_charge_capacity → charge_capacity`; `test_time`, `step_time`, `current`, `internal_resistance` keep their names |
| `STEP_BASE_PAIRS` × `STAT_SUFFIXES` | 8 × 7 | base signals expanded per statistic: `potential_mean → voltage_avr` (only `mean → avr` differs among the suffixes), `charge_capacity_last → charge_last`, `datapoint_num_first → point_first`, `internal_resistance_delta → ir_delta` |
| `STEP_SCALAR_PAIRS` | 6 | `cycle_num → cycle`, `step_num → step`, `sub_step_num → sub_step`, `step_type → type`, `sub_step_type → sub_type`, `c_rate → rate_avr` |
| `CYCLE_PAIRS` | 23 | `last_test_time → test_time`, `datapoint_num_last → data_point`, `test_cumulated_* → cumulated_*`, `potential_end_charge → end_voltage_charge`, `temperature_cell_mean → temperature_mean`, plus **5 identity pass-throughs** (`ir_charge`, `ir_discharge`, `charge_c_rate`, `discharge_c_rate`, `normalized_cycle_index`) declared so the totality claim holds |

### 2.2 The exception sets ("lossless and total, modulo documented exceptions")

Columns with no counterpart on the other side are listed **explicitly** (not derived):

| Set | Size | Content |
|---|---|---|
| `LEGACY_ONLY_RAW` | 19 | `date_time`, `power`, `charge_energy`, `dv_dt`, `sub_step_index/-time`, AC-impedance columns, `test_name`, … |
| `NATIVE_ONLY_RAW` | 19 | `epoch_time_utc`, `mask`, `source_*`, `step_type`/`step_mode`/`cycle_type`, cumulative energies, step powers, aux channels, `ref_potential` (deliberately **not** bridged to legacy `reference_voltage` — would grow the legacy step frame and break byte parity, issue #43) |
| `LEGACY_ONLY_STEP` | 4 | `test`, `ustep`, `info`, `ir_pct_change` |
| `NATIVE_ONLY_STEP` | 6 | `power`, `charge_energy`, `discharge_energy`, `mask`, `test_id`, `ref_potential` (base-signal granularity) |
| `LEGACY_ONLY_CYCLE` | 19 | `cumulated_coulombic_efficiency`, `shifted_*`, `cumulated_ric*`, `ocv_*`, `normalized_*_capacity`, `low/high_level`, … |
| `NATIVE_ONLY_CYCLE` | ~68 | durations, energies, per-direction current/potential/power statistics, CV/CC split, epoch times, `test_id`, `mask`, … |

`test_id` exists on both sides with the same name but is *intentionally not bridged*
(dropped from all three legacy frames), so it appears in the exception sets on both
sides.

### 2.3 Derivation helpers

The bridge never writes a rename dict by hand; it calls:

- `legacy_to_native_raw(columns)` — filtered to the columns actually present, so it is
  safe to pass straight to `DataFrame.rename`
- `native_to_legacy_step()` / `legacy_to_native_step()` — `STEP_BASE_PAIRS` expanded
  with every `STAT_SUFFIXES` variant + the scalar pairs
- `native_to_legacy_summary()` / `legacy_to_native_summary()`

## 3. The bridge: `OldCellpyCellCore`

`CellpyCell.__init__` creates the bridge once
(`self.core = OldCellpyCellCore(initialize=False)`, cellreader.py:249). The bridge
subclasses the native `CellpyCellCore` and adds three legacy-shaped entry points.
Each follows the same sandwich: **rename legacy→native → polars engine →
rename native→legacy → restore legacy quirks**.

### 3.1 `make_core_step_table(data)` — called from `CellpyCell.make_step_table` (cellreader.py:3075)

1. `data.raw.rename(columns=legacy_to_native_raw(raw.columns))`
2. `pl.from_pandas(...)` → native polars raw frame
3. Native engine: `summarizers.make_step_table` (+ the C-rate expression appended
   *before* the rename so `rate_avr` lands in its byte-for-byte legacy position)
4. `.to_pandas()` + `rename(columns=native_to_legacy_step())`
5. Reorder to the exact legacy `HeadersStepTable` column order
   (`_legacy_step_column_order`)
6. Legacy sorting (`sort_values` + `reset_index` reproducing the legacy `index`
   column) and, if step specifications were given, the `info` column is mapped in
   pandas so `NaN` keeps its legacy `str(NaN) == "nan"` behaviour

cellpy keeps orchestration on its side of the seam: nominal-capacity resolution to an
absolute value, `raw_limits` passed by value, deprecation handling.

### 3.2 `make_core_summary(data)` — called from `CellpyCell._make_summary` (cellreader.py:5908)

1. **Both** `data.raw` and `data.steps` are renamed legacy→native and converted to
   polars
2. Native engines: `make_summary`, then `ir_to_summary` and `c_rates_to_summary` —
   these produce columns whose native names *equal* the legacy names (issue #21), so
   they survive the later rename untouched
3. `.to_pandas()` + `rename(columns=native_to_legacy_summary())`
4. `date_time` is merged back from the **legacy raw** frame keyed on `data_point`
   (the native raw carries `epoch_time_utc`, not `date_time`)
5. `_add_legacy_summary_cruft` computes, in pandas, the legacy-only columns the
   curated native schema deliberately omits: `cumulated_coulombic_efficiency`,
   `shifted_charge/discharge_capacity`, `cumulated_ric*`
6. Reorder to the legacy summary column order

cellpy pre-computes `current_conversion_factor` and the per-mode
`specific_converters` with pint and passes them in as plain numbers.

### 3.3 `add_scaled_summary_columns(data)` — called right after (cellreader.py:5915)

1. `data.summary.rename(columns=legacy_to_native_summary())` → polars
2. Native `equivalent_cycles_to_summary` + `generate_specific_summary_columns`
   produce `{col}_{mode}` columns for each mode (`gravimetric`, `areal`, `absolute`)
3. The native→legacy rename dict is **extended on the fly**:
   `f"{col}_{mode}" → f"{legacy_col}_{mode}"` — this is how the legacy postfix
   convention (`charge_capacity_gravimetric`, …) is reproduced
4. Back to pandas as the legacy summary

## 4. Guarantees (`cellpy-core/tests/test_header_mapping.py`)

- **Bijectivity** — `STAT_SUFFIXES` and every pair list are one-to-one
- **Round-trip identity** — legacy→native→legacy (and the reverse) is the identity
  for step and summary rename dicts
- **Totality** — for each of the six class/side combinations: declared columns ==
  (mapped columns) ∪ (that side's exception set). Adding a new column on either side
  **fails the test until it is deliberately categorised** as mapped or exception
- **Known translations** — spot checks of the tricky renames (`potential`/`voltage`,
  `mean`/`avr`, …)
- **Bridge parity** — the rename dicts the bridge actually uses are exactly the
  mapping-derived ones (the bridge holds no header knowledge of its own)

## 5. Properties worth knowing (and watching)

1. **The swap is total and tested, but only at the seam.** Everything inside cellpy
   still speaks legacy names (including the hard-coded literals catalogued in
   [hardcoded-column-headers-report.md](hardcoded-column-headers-report.md)); the
   bridge shields the core from that entirely.
2. **Byte-for-byte legacy output is a design goal** ("the golden oracle") — column
   order, the `index` column from legacy sorting, `"nan"` strings in `info`, and the
   postfix columns are all reproduced exactly.
3. **Legacy cruft lives in the bridge, not the core.** Shifted capacities, RIC,
   cumulated CE are computed in pandas in `_add_legacy_summary_cruft`; the native
   `CycleCols` schema stays clean.
4. **Native-only richness is invisible to cellpy.** Energies, powers, durations,
   per-direction statistics and epoch timestamps exist only on the native side; the
   legacy frames never see them. Migrating a util to the native schema is what
   unlocks them.
5. **Two copies of the legacy headers exist** — `cellpy/parameters/internal_settings.py`
   and the mirror in `cellpycore/legacy/headers.py`. They must stay in sync manually;
   the parity tests catch value drift only for the *mapped* columns.
6. **`date_time` is a special case** — the only column that is re-joined from the
   input frame after the engine run rather than translated.
