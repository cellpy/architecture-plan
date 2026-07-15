# Migration notes: cellpy 1.0.3 → 1.0.4 (cellpycore engine)

*Draft, 2026-07-15. Companion to `cellpy-v103-vs-v104a3-observations.md`, which
records the full validation and the cellpy-core issue/PR trail. This note is the
user-facing subset: what changes in your results and scripts.*

Raw data, capacities, voltages, IR, C-rates and the cumulated-capacity columns
are unchanged (verified to `rtol 1e-6`). The items below are the only behavioural
or schema differences a user or downstream script will notice.

## 1. Set `cycle_mode="anode"` explicitly for anode half-cells (action required)

The cellpycore engine defaults `cycle_mode` to the **normal** (full-cell /
cathode) convention. Legacy cellpy historically defaulted to `"anode"`. If you
test anode half-cells, **set `cycle_mode="anode"` explicitly** (as you load /
configure the cell) — otherwise `coulombic_efficiency` and the
`coulombic_difference` family come out with the wrong direction (first-cycle CE
appears above 100 % instead of below, sign of `coulombic_difference` flipped).

With `cycle_mode="anode"` set, these columns match cellpy 1.0.3 exactly. (The
engine now both recognises the mode string robustly and applies it on every code
path — cellpy-core #127 and #129.)

## 2. Dropped summary / step columns (intentional)

These columns are no longer produced. Old `.h5` / `.cellpy` files that contain
them still load; the new engine simply does not regenerate them. None are read by
cellpy's own analysis or plotting code, so this only affects external scripts that
referenced the exact column names.

- **Step table:** the seven `reference_voltage_*` aggregates
  (`_avr`, `_std`, `_min`, `_max`, `_first`, `_last`, `_delta`). The raw
  `reference_voltage` signal is unchanged.
- **Summary:** the six *specific* shifted-capacity variants —
  `shifted_charge_capacity_{gravimetric,areal,absolute}` and the discharge
  equivalents. The **unscaled** `shifted_charge_capacity` /
  `shifted_discharge_capacity` columns remain; scale them yourself if you need
  per-mass / per-area values.

## 3. `sub_type` is now a real null, not the string `"None"`

The step table's `sub_type` column now stores a real null (`None` / `NaN`) where
1.0.3 stored the literal string `"None"`. Update any checks like
`sub_type == "None"` to `sub_type.isna()`. CSV exports now show an empty cell
instead of `None`.

## 4. One extra `cv_charge` step in some files (bug fix)

1.0.3 occasionally left a constant-voltage step unclassified (empty `type`) with a
corrupted, negative duration, because its aggregates were computed from
wrongly-ordered rows. The new engine classifies and times it correctly (e.g.
cycle 9 / step 21 in the SUMBATSP2 anode file becomes `cv_charge`). Code that
groups steps by `type` may see one additional correctly-typed step in affected
files. This is a fix, not a regression.
