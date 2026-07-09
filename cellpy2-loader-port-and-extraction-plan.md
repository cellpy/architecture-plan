# Plan: instrument-loader port to harmonized raw + the extraction layer (G2 + G4)

**Date:** 2026-07-09
**Closes:** gap-analysis items **G2** (loader port) and **G4** (extraction layer),
and absorbs **F3** (potential/seconds), **F5** (reset granularity #42,
`ref_potential` #43), and the loader passes scheduled by the
[unit plan](unit-handling-cellpy2-plan.md) (Phase 3) and the
[metadata plan](cellpy2-metadata-handling-plan.md) (Step 3) — **one pass over the
loaders, not three**.

---

## 1. Current state

### 1.1 The loader framework (`cellpy/readers/instruments/`)

- `base.py`: `AtomicLoad` → `BaseLoader` → `AutoLoader` → `TxtLoader` class tower;
  `loader_executor` orchestrates; loaders return a legacy `Data` with:
  - raw frame in **legacy `HeadersNormal`** names, `data_point` promoted to index
    (polars report §1.1 — must die),
  - `raw_units` as ad-hoc dicts (163 hard-coded unit-key literals across
    `configurations/*`),
  - metadata poked directly onto `Data` attributes (`data.start_datetime = …`).
- 16 loaders in three tiers of health:
  | Tier | Loaders | Notes |
  |---|---|---|
  | 1 — canonical, testdata in-tree | `arbin_res`, `maccor_txt`, `neware_txt`, `pec_csv`, `custom` | port first; goldens exist or are cheap |
  | 2 — SQL/export family | `arbin_sql`, `arbin_sql_7`, `arbin_sql_csv`, `arbin_sql_h5`, `arbin_sql_xlsx`, `neware_xlsx`, `neware_nda` | shared vendor semantics with tier 1 |
  | 3 — exotic / plugin-ish | `biologics_mpr`, `batmo_bdf`, `ext_nda_reader` (flagged not-prod-ready), `local_instrument` | decide keep/port/park per loader |
- `configurations/*` modules declare vendor→legacy renaming dicts, unit dicts,
  post-processing hooks.

### 1.2 What cellpy-core already provides (build on, don't duplicate)

- **The authoritative target schema**: `docs/data_format_specifications/harmonized_raw.md`
  — native `RawCols` incl. `potential` (not voltage), `test_time` in **seconds**,
  `epoch_time_utc` (int64 ns UTC, `cellpycore.timestamps`), `mask`, `test_id`,
  provenance (`source_type/uuid/datapoint_num/step_num`), `step_mode`, `cycle_type`,
  cumulative energies, `aux_<quantity>_<name>` scheme.
- `RawCols.dtype_map()` (issue #97) — the column→polars-dtype single source of truth.
- `validate_raw_frame(raw, raw_cols)` (`cell_core.py:31`) and
  `Data.from_raw_frame(...)` (`cell_core.py:164`) — the ingestion door.
- `StepType`/`StepMode`/`CycleType` StrEnums (issue #24) as reference vocabularies.

## 2. Target design

### 2.1 Two stages: vendor parsing (keep) + shared `harmonize()` (new)

Loaders keep their vendor-specific parsing (the hard-won part). Everything after
parsing moves into **one shared normalization stage** owned by the framework:

```
vendor file ──loader.parse()──► vendor frame + declarations ──harmonize()──► native raw
                                                             (framework-owned)
```

`harmonize()` responsibilities, driven by *declarations* in each loader's
configuration (not code):

1. **Rename** vendor columns → native `RawCols` names (configurations declare
   `vendor_name -> native_name`; the legacy `*_txt` indirection dies).
2. **Cast** via `RawCols.dtype_map()`.
3. **Timestamps**: vendor datetime → `epoch_time_utc` int64 ns UTC
   (`cellpycore.timestamps`); per-loader timezone declaration (naive-datetime rule
   shared with the header-migration plan's D3 decision).
4. **Reset-granularity normalization** (core issue #42): each configuration declares
   whether capacity/energy columns are per-step, per-cycle, or per-test cumulative;
   `harmonize()` normalizes to the spec'd convention. *This is the correctness
   landmine of the whole port — see §5 tests.*
5. **Units**: configuration declares `raw_units` as a validated `CellpyUnits`
   (unit plan Phase 3: pint-parsable check at load, floats banned).
6. **Identity/provenance**: `test_id = 0`, `mask = True`,
   `source_type/uuid/datapoint_num/step_num` filled by the framework;
   `source_uuid` = stable hash of file identity (feeds `TestMeta`).
7. **No index**: `data_point` stays a column (`datapoint_num`), never an index.
8. **Validate**: `validate_raw_frame` runs on every load in strict mode
   (warn-only escape hatch for `local_instrument`).
9. **Metadata**: loader returns draft `TestMeta` (+ partial `CellMeta` when the file
   carries masses etc.) per the metadata plan's Step-3 contract; the framework fills
   provenance fields.

`ref_potential` (core issue #43): loaders that have a reference electrode declare the
column; it flows on the native path only (never bridged to legacy — already decided).

### 2.2 Configuration format

Extend the `configurations/*` modules (or their YAML successors — coordinate with the
config plan's instrument models) to declare, per instrument/model:
`column_map` (vendor→native), `units` (CellpyUnits), `tz` rule, `reset_granularity`
per cumulative column, `aux_map` (vendor aux channels → `aux_<quantity>_<name>`),
optional `post_hooks`. One schema, validated at registration time — a typo'd
declaration fails at import of the configuration, not mid-load.

### 2.3 The extraction layer (G4 decision)

**Decision proposed: a `cellpycore.curves` module.** `get_cap` / `get_ccap` /
`get_dcap` / `get_ocv` / `collect_capacity_curves` are pure frame computations over
raw+steps — exactly what core exists for; keeping them in cellpy 2 would leave the
most-used data API on the wrong side of the seam and the curve-frame schema
unspecified forever (headers report §5).

- **Spec first**: `docs/data_format_specifications/curve_table.md` + a `CurveCols`
  class (`cycle_num`, `capacity`, `potential`, `direction`, optionally `test_id`,
  `mode`) — mirrors how `extractors.py` was introduced (issue #23 pattern).
- Port the selection logic (interpolation, taper trimming, forth-and-forth modes,
  steptable override) polars-native, schema-injected.
- cellpy 2 `CellpyCell.get_cap(...)` becomes a thin wrapper (units/labels applied at
  the app layer); legacy column names for the returned frame (`"voltage"`,
  `"capacity"`) available through the same deprecation shim as everything else.
- The utils that consume curves (easyplot, collectors, ica, plotutils) migrate in the
  [utils plan](cellpy2-utils-migration-plan.md) wave 2/3.

## 3. Migration steps

| Step | Content | Depends on |
|---|---|---|
| 0 | **Golden per-loader fixtures**: for each tier-1/2 loader, commit a small vendor file (or reuse testdata) + the expected harmonized parquet (regenerated by script, per the shared fixture convention F8) | — |
| 1 | `harmonize()` + configuration schema + `validate_raw_frame` wiring; port **one** pilot (recommend `maccor_txt` — TxtLoader family, rich configurations) end-to-end | core: nothing new |
| 2 | `CurveCols` spec + `cellpycore.curves` port + parity tests vs legacy `get_cap` on goldens | core release (F9: core-first merge order) |
| 3 | Tier-1 loaders onto `harmonize()` (arbin_res, neware_txt, pec_csv, custom) | 1 |
| 4 | Tier-2 loaders; shared vendor semantics factored into family configs | 3 |
| 5 | Tier-3 decisions: port `biologics_mpr` + `batmo_bdf`; **park** `ext_nda_reader` (already flagged non-prod; `local_fastnda` lib exists) unless users object; `local_instrument` gets the warn-only validation mode | 4 |
| 6 | Delete legacy normalization from `base.py` (the `*_txt` rename path, index promotion); loaders now emit native-only | header-migration Phase 3 |

Each loader PR = configuration declarations + fixture + parity test; the framework
changes land once (Steps 1–2).

## 4. What this plan absorbs from the others

- **Unit plan Phase 3** — `raw_units` validation, float ban, v7 conversion quarantine:
  implemented inside `harmonize()`/configuration schema (Steps 1, 3–5).
- **Metadata plan Step 3** — loader draft-`TestMeta` contract + `MetaResolver`
  boundary: the framework side lands in Step 1, per-loader drafts in Steps 3–5.
- **F3** — `potential`/seconds normalization is Step 1's rename/cast stage.
- **F5** — reset-granularity declarations (Step 1 schema), `ref_potential`
  declaration (per-loader).

## 5. Test plan

- Per-loader golden: vendor file → `harmonize()` → frame-equality vs committed
  parquet (dtype-exact, incl. `epoch_time_utc` values).
- **Reset-granularity property test**: recomputed per-cycle capacities from
  normalized raw must equal the legacy pipeline's summary capacities on the golden
  cells (this is the test that catches a wrong granularity declaration).
- `validate_raw_frame` strictness: an undeclared vendor column or wrong dtype fails
  the loader's own test.
- Curves parity: `cellpycore.curves` vs legacy `get_cap` on goldens (values, per
  mode/direction), plus NullData edge cases (empty cycles).
- End-to-end: harmonized raw → native `make_step_table`/`make_summary` → value parity
  with legacy pipeline (the header-migration Phase-3 oracle covers this once loaders
  feed it).

## 6. Risks

| Risk | Mitigation |
|---|---|
| Wrong reset-granularity declaration silently corrupts capacities | The §5 property test per loader; declarations reviewed against vendor docs |
| Timezone semantics per vendor (Arbin local vs Neware device-time) | Per-loader `tz` declaration, defaulting to the shared naive-datetime rule; recorded on `TestMeta.time_zone` |
| ODBC/Access dependency for `arbin_res` on CI | Keep the committed harmonized fixture as oracle (the core testdata approach — F8); loader test runs where the driver exists, fixture parity everywhere |
| Curves port drifts from legacy behavior in interpolation corner cases | Parity fixtures over multiple modes; document intentional differences (there are known quirks in forth-and-forth) |
| Three plans' loader changes colliding | This plan **is** the merge point; unit/metadata plans reference it instead of scheduling their own loader passes |

## 7. Effort

Framework + pilot ≈ 4–5 days; curves module ≈ 3–4 days (incl. spec); tier-1 ≈ 1 day
per loader; tier-2 ≈ 3–4 days total; tier-3 decisions ≈ 1–2 days. Total ≈ **3–4
focused weeks**, parallelizable per loader after Step 1.
