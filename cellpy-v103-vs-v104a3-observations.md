# Observed behavior differences: cellpy 1.0.3 vs 1.0.4a3 (and cellpycore)

*Date: 2026-07-09.
Produced by the validation notebooks in `cellpy-core-dev/notebooks/` (marimo).
Reference data computed by real cellpy 1.0.3 in an isolated environment
(python 3.12, pandas 2.3), frozen to parquet in
`cellpy-core-dev/data/reference_v103/`; the "new" side is cellpy 1.0.4a3
(python 3.14, pandas 3) plus the cellpycore polars engine.*

Detailed numbers below come from `20231115_SUMBATSP2_GC2_15_cc.h5`
(anode half-cell, 21 cycles); the notebooks run the same comparison for all
six test files.

## TL;DR

| Area | Verdict |
|------|---------|
| raw data | **identical** between 1.0.3 and 1.0.4a3 |
| step table | numerically identical except **one step that 1.0.3 misclassified** (bug fixed in new version); `sub_type` representation changed; `reference_voltage_*` aggregate columns dropped |
| summary | `coulombic_difference` family **flipped sign**; `coulombic_efficiency` **inverted** (reciprocal); `shifted_*_capacity` changed accordingly; six `shifted_*` specific columns dropped |
| cellpycore engine vs legacy | step numbers identical (only `type`/`sub_type` label conventions differ); summary differs in the same coulombic columns plus `discharge_c_rate` |

None of these look like accidental numerical drift — they are discrete,
systematic convention changes (sign, ratio direction, schema) plus one clear
bug fix. Each should be confirmed as *intended* and, where user-facing,
mentioned in release notes / migration docs.

## Resolution status (2026-07-15)

All deltas below have been triaged against the current cellpy-core code and
resolved. Summary (details in each section and in the linked cellpy-core issues):

| Delta | Verdict | Action taken |
|-------|---------|--------------|
| §3.1 sign flip + §3.2 CE inversion | **Regression, now fixed** | Root cause was `cycle_mode="anode"` not being applied as `TestMode.INVERTED`: the string→enum translator only matched the literal `"anode"` (cellpy-core #127) *and* the legacy bridge summary never passed `test_mode` (cellpy-core #129). Both merged; anode CE/coulombic_difference now match 1.0.3. |
| §2.1 misclassified step | **1.0.3 bug, correctly fixed in new** | Accepted as an improvement; added a guard test that `step_time_delta` is never negative (cellpy-core #132). |
| §4 `discharge_c_rate` | **Not a bug / not reproducible** | Both engines agree exactly on charge and discharge C-rate at any `nom_cap`; the reported mismatch was a notebook nom-cap plumbing artifact (or already fixed by the C-rate refactor #21/#98/#99). Closed (cellpy-core #130). |
| §2.2 `sub_type` null, §2.3 `reference_voltage_*`, §3.3 `shifted_*` specific | **Intended; keep as-is** | Consumer grep across the cellpy repo found **none** of these are read by cellpy analysis/plotting code (cellpy-core #131). Documented as intentional; old files still load. See `cellpy-v104-migration-notes.md`. |

## 1. Raw data — identical

All 16 raw columns agree exactly (rtol 1e-6) for all aligned data points.
File reading of legacy `.h5` files is unaffected by the pandas 2 → 3 and
python 3.12 → 3.14 jumps.

## 2. Step table

### 2.1 One misclassified step in 1.0.3 (bug, fixed)

Cycle 9, step 21 (130 raw points):

| | type | current_first (A) | step_time_delta (s) |
|---|---|---|---|
| v1.0.3 | *(empty)* | 0.02237 | **−0.00046** |
| v1.0.4a3 | `cv_charge` | 0.00099 | 1986.9 |

1.0.3 gave the step an empty type and a *negative* duration — its
first/last/delta aggregates were computed from wrongly ordered points. The
new version classifies and aggregates it correctly. This is the only step
(of 151) whose numbers differ; it is an **improvement**, not a regression,
but any downstream code that groups on step `type` will see one extra
`cv_charge` step in this file.

### 2.2 `sub_type` representation

1.0.3 stores the string `"None"`; 1.0.4a3 stores a real NaN/None. Cosmetic,
but string-comparison-based code (or round-trips through CSV) will notice.

### 2.3 Dropped columns

The seven `reference_voltage_*` aggregate columns (`_avr`, `_std`, `_min`,
`_max`, `_first`, `_last`, `_delta`) exist in 1.0.3 step tables but not in
1.0.4a3, even though the raw table still has `reference_voltage`. Intended?

## 3. Summary

### 3.1 Sign flip: `coulombic_difference` family

Exact sign flip (relative difference is exactly 2.0 on every cycle):

| cycle | v1.0.3 | v1.0.4a3 |
|---|---|---|
| 1 | +0.000322 | −0.000322 |
| 2 | +0.000053 | −0.000053 |
| 4 | −0.000402 | +0.000402 |

Affects `coulombic_difference`, `cumulated_coulombic_difference` and all
their `_gravimetric` / `_areal` / `_absolute` variants, and consequently
`shifted_charge_capacity` / `shifted_discharge_capacity` and
`cumulated_ric*`. Presumably the definition changed from
`charge − discharge` to `discharge − charge` (or vice versa).

### 3.2 Coulombic efficiency inverted (reciprocal)

New value = 1 / old value — the ratio direction flipped
(e.g. charge/discharge → discharge/charge), presumably tied to anode
cycle-mode handling:

| cycle | v1.0.3 (%) | v1.0.4a3 (%) |
|---|---|---|
| 1 | 94.31 | 106.03 |
| 3 | 99.24 | 100.76 |
| 4 | 107.46 | 93.05 |

For a **plot**, this is very visible: first-cycle CE moves from below 100%
to above 100%. Anyone with CE-based QC thresholds (e.g. "first-cycle CE >
90%") will get different answers. This is the single most user-facing
change found.

### 3.3 Dropped columns

Gone in 1.0.4a3: `shifted_charge_capacity_gravimetric`,
`shifted_discharge_capacity_gravimetric`, `shifted_charge_capacity_areal`,
`shifted_discharge_capacity_areal`, `shifted_charge_capacity_absolute`,
`shifted_discharge_capacity_absolute`. (The unscaled `shifted_*` columns
remain, with the sign change from 3.1.)

### 3.4 Everything else agrees

Capacities (raw, gravimetric, areal), end voltages, IR, c-rates and the
cumulated capacity columns match within rtol 1e-6. Capacity-vs-cycle and
voltage-capacity **graphs overlay exactly** between versions.

## 4. cellpycore (polars) engine vs legacy engine

Legacy raw bridged to the native schema
(`cellpycore.legacy.mapping.legacy_to_native_raw`), run through
`CellpyCellCore.make_core_step_table` / `make_core_summary`, mapped back to
legacy names:

- **Steps:** all shared numeric columns identical; only `type`/`sub_type`
  label conventions differ (clean one-to-one relabelling; verify with the
  crosstab in notebook 06).
- **Summary:** differs in `coulombic_difference`,
  `cumulated_coulombic_difference`, `coulombic_efficiency` (consistent with
  §3 — the new conventions) and `discharge_c_rate` (worth a look: is the
  nominal-capacity input to the C-rate calculation equivalent in both
  engines?).
- Native summary is much narrower (21 columns vs 51+); scaled/specific
  columns come from `add_scaled_summary_columns` as an explicit second step.

## 5. Open questions — resolved (2026-07-15)

1. Are the sign flip (§3.1) and the CE inversion (§3.2) intended convention
   changes? **No — both were a regression, now fixed.** They were two faces of
   one root cause: `cycle_mode="anode"` was not applied as `TestMode.INVERTED`.
   Fixed in cellpy-core #127 (translator only matched the literal `"anode"`) and
   #129 (legacy bridge summary never passed `test_mode`). Anode
   `coulombic_efficiency` / `coulombic_difference` now match 1.0.3.
2. Should `reference_voltage_*` step aggregates (§2.3) and the
   `shifted_*` specific summary columns (§3.3) come back? **No.** A consumer grep
   across the cellpy repo found neither is read by analysis/plotting code
   (cellpy-core #131). Kept dropped; old files still load. Documented in
   `cellpy-v104-migration-notes.md`.
3. `discharge_c_rate` mismatch between legacy and cellpycore engines (§4) —
   nominal-capacity plumbing or rounding? **Neither — not reproducible.** Both
   engines agree exactly (charge and discharge, any `nom_cap`); the mismatch was
   a notebook nom-cap plumbing artifact. Closed (cellpy-core #130).
4. `sub_type` `"None"`-string vs NaN (§2.2) — worth normalizing in the
   legacy bridge? **No.** No cellpy code string-compares `sub_type` against
   `"None"`; the native engine's real null is kept. Documented as a migration
   note for external CSV consumers.

## Reproducing

```bash
cd cellpy-core-dev

# regenerate the frozen 1.0.3 reference (isolated env)
uv run --isolated --no-project --python 3.12 \
    --with cellpy==1.0.3 --with pyarrow \
    python scripts/make_reference_v103.py

# explore the comparisons interactively
uv run marimo edit notebooks/05_regression_v103_vs_new.py   # 1.0.3 vs current
uv run marimo edit notebooks/04_stored_vs_recomputed.py     # stored vs recomputed
uv run marimo edit notebooks/06_cellpycore_engine_check.py  # legacy vs polars engine
```
