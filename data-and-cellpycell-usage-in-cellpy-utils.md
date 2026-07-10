# Usage of `Data` and `CellpyCell` in `cellpy/utils`

**Date:** 2026-07-08
**Scope:** All `*.py` files under `cellpy/cellpy/utils` (including `batch_tools/`).
**Classes analyzed:**

- `Data` — `cellpy/readers/data_structures.py`
- `CellpyCell` — `cellpy/readers/cellreader.py`

**Method:** The public API (properties, methods, and instance attributes assigned in
`__init__`) of both classes was extracted with AST parsing. Every `.member` access in the
utils files was then matched and classified by inspecting the receiver expression, so that
false positives (e.g. `logging.debug`, `str.split`, `DataFrame.empty`, `fig.data` from
plotly, `hdr_journal.nom_cap` journal-header lookups, `json.load`) are excluded.
References that occur only in docstrings/examples are marked *(doc-only)*.

---

## 1. `CellpyCell` members used by the utils

### 1.1 Properties

| Member | Used in | Notes |
|---|---|---|
| `data` | batch.py, batch_core.py, batch_experiments.py, batch_analyzers.py, engines.py, easyplot.py, helpers.py, ocv_rlx.py, plotutils.py, collectors.py | **The single most important member.** Nearly all `Data` access from utils goes through `c.data.…` (~100+ real call sites). |
| `empty` | batch.py:418, batch_analyzers.py:147–182, batch_experiments.py:468, 802, helpers.py:810, 1150, ica.py:873 | Guard before processing a cell. |
| `cell_name` | batch.py:275, collectors.py:1605–1713, plotutils.py:1599, 2407, 3353, 5163, 5461, 5764 | Titles/labels and log messages. |
| `mass` | helpers.py:1604 | Setter use (`d.mass = mass`). |
| `active_mass` | batch.py:278 | Read when building journal pages. |
| `tot_mass` | batch.py:279 | Read when building journal pages. |
| `nominal_capacity` | batch.py:281 | Read when building journal pages. |
| `nom_cap_specifics` | batch.py:288 | Read when building journal pages. |
| `raw_units` | collectors.py:568, plotutils.py:5147–5157 | Forwards to `Data.raw_units` (utils also access it directly via `c.data.raw_units`, see §2). |

### 1.2 Instance attributes (set in `__init__`)

| Member | Used in | Notes |
|---|---|---|
| `cellpy_units` | collectors.py:563, plotutils.py:603–606, 821–836, 5971, 6030, 6046 | Heavily used for axis/label unit strings. |
| `headers_summary` | plotutils.py:817, 959, 1324, 1374, 1494, 1518 | Column-name lookups for summary plots. |
| `headers_step_table` | ocv_rlx.py:364 | Via `self.data.headers_step_table` (`self.data` holds a `CellpyCell` there). |
| `ensure_step_table` | batch_experiments.py:199, 527, 852 | **Written** (`= True`) by batch tools to force step-table creation. |
| `overwrite_able` | batch_core.py:344, 355 | **Written** (`= False`) before loading cellpy-files. |
| `last_uploaded_from` | batch.py:267 | Read when building journal pages. |

### 1.3 Methods

| Member | Used in | Notes |
|---|---|---|
| `get_cap` | easyplot.py:568, 779, 869, 1143, 1283, 1406, collectors.py:1616, ica.py:913, plotutils.py:5784 | Most-used data-extraction method. Also in docstrings: batch_core.py:386, collectors.py:1371, ica.py:472, 602 *(doc-only)*. |
| `get_ccap` | batch_helpers.py:449, ica.py:1033 | |
| `get_dcap` | batch_helpers.py:460 | |
| `get_ocv` | ocv_rlx.py:305, 325, 631 | |
| `get_cycle_numbers` | batch_helpers.py:444, collectors.py:1608, 1689, easyplot.py:275, ica.py:1106, ocv_rlx.py:75, 846, 938, plotutils.py:5756 | Widely used across utils. |
| `make_summary` | batch_experiments.py:492, 817, 1089, 1091, helpers.py:917, 920, 1156, 1606, plotutils.py:1235, 3434, 4970, 4973 | |
| `make_step_table` | batch_experiments.py:486, 813, 1083, 1085, helpers.py:1154, 1605 | |
| `save` | batch_experiments.py:200, 529, 854, 937, 987, 1102, example_data.py:299, helpers.py:1607 | |
| `load` | batch_core.py:357, ica.py:1105 | |
| `from_raw` | helpers.py:1603 | |
| `to_csv` | batch_experiments.py:557, 879, helpers.py:1608 | |
| `drop_from` | helpers.py:812, 1152 | Used for trimming cycles (note: returns a new object, `c = c.drop_from(...)`). |
| `has_no_partial_duplicates` | batch.py:344 | |
| `get_converter_to_specific` | helpers.py:266 | |
| `unit_scaler_from_raw` | plotutils.py:5307–5309 | |
| `set_mass` | batch_experiments.py:1078 | |
| `set_nom_cap` | batch_experiments.py:1075 | |
| `vacant` | helpers.py:204 *(doc-only — mentioned in a deprecation-warning message)* | |
| `filtered_summary` | plotutils.py:4823, 4825 *(doc-only — docstring example)* | |

---

## 2. `Data` members used by the utils

Utils never construct `Data` directly; every access goes through `CellpyCell.data`
(receivers such as `c.data.…`, `cell.data.…`, `cpobj.data.…`, `cell_data.data.…`).

### 2.1 Attributes (the three DataFrames dominate)

| Member | Used in | Notes |
|---|---|---|
| `summary` | batch.py:370–408, 623, batch_experiments.py:479, 808, engines.py:121, easyplot.py:633, helpers.py (~20 sites: 279–283, 497–656, 838, 915–922, 1192, 1476, 1547–1566), plotutils.py (1230, 1313, 3428, 3572, 4830, 4934, 4968–4975) | **Most-accessed `Data` member** (~40 real sites). Both read and written (helpers.py modifies/normalizes summaries). |
| `steps` | batch.py:362, 426, 631, batch_core.py:346, batch_experiments.py:482, 516, 811, 841, easyplot.py:653, 1481, helpers.py:267–268, 1471, ocv_rlx.py:85, 363, plotutils.py:5368, 5531 | Read for step-type filtering and cycle info. |
| `raw` | batch.py:354, 627, batch_core.py:226, ocv_rlx.py:86, plotutils.py:5050, 5367, 5530 | |
| `raw_units` | plotutils.py:833, 5061–5141 (15 sites) | Unit labels for raw-data plots. |
| `meta_common` | batch.py:283 | Whole meta object read when building journal pages. |
| `raw_data_files` | batch.py:289 | |
| `loaded_from` | batch_helpers.py:433 | |

### 2.2 Properties (meta-data forwarders)

| Member | Used in | Notes |
|---|---|---|
| `nom_cap` | helpers.py:264, 816, 1160, plotutils.py:1499 (+ docstrings 1480, 4743–4744, 4812) | Used as rescale factor for rate calculations. |
| `cell_name` | batch.py:282, 287 | |
| `loading` | batch.py:280 | |

### 2.3 `Data` members NOT used by any util

`active_electrode_area`, `custom_info`, `has_data`, `has_steps`, `has_summary`,
`empty` (only reached indirectly through `CellpyCell.empty`), `mass`, `tot_mass`,
`material`, `start_datetime`, `raw_id`, `meta_test_dependent`, `populate_defaults`,
`raw_data_files_length`, `raw_limits`.

*(Several of these — `mass`, `tot_mass`, `empty`, `cell_name`, `nom_cap` — are reached
indirectly because `CellpyCell` forwards its same-named properties to `self.data`.)*

---

## 3. `CellpyCell` public members NOT used by any util

Grouped by theme (useful when deciding what must survive a core refactor):

- **Data extraction:** `get_raw`, `get_voltage`, `get_current`, `get_datetime`,
  `get_timestamp`, `get_ir`, `get_summary`, `get_number_of_cycles`, `get_rates`,
  `get_step_numbers`, all `sget_*` methods (`sget_voltage`, `sget_current`,
  `sget_steptime`, `sget_timestamp`, `sget_step_numbers`)
- **Slicing/splitting:** `split`, `split_many`, `from_cycle`, `to_cycle`, `drop_to`,
  `drop_edges`, `with_cycles`, `mod_raw_split_cycle`
- **IO/config:** `loadcell`, `merge`, `to_excel`, `to_bdf`, `set_instrument`,
  `set_raw_datadir`, `set_cellpy_datadir`, `register_instrument_readers`,
  `check_file_ids`, `load_step_specifications`, `initialize`
- **Steps/summary manipulation:** `select_steps`, `populate_step_dict`, `print_steps`,
  `add_to_summary`, `inspect_nominal_capacity`
- **Units/quantities:** `with_cellpy_unit`, `to_cellpy_unit`,
  `nominal_capacity_as_absolute`, `total_time_at_voltage_level`
- **Properties/attributes:** `active_electrode_area`, `nom_cap` (utils go via
  `c.data.nom_cap` instead), `cycle_mode`, `get_mass`, `set_tot_mass`,
  `has_data_point_as_index`, `has_data_point_as_column`, `has_no_full_duplicates`,
  `set_col_first`, and most config-ish attributes (`auto_dirs`, `capacity_modifiers`,
  `cellpy_datadir`, `raw_datadir`, `tester`, `profile`, `select_minimal`,
  `limit_data_points`, `limit_loaded_cycles`, `ensure_summary_table`,
  `force_step_table_creation`, `force_all`, `sep`, `headers_normal`, …)

---

## 4. Per-module dependency overview

| Util module | CellpyCell surface used | Data surface used |
|---|---|---|
| `batch.py` | `data`, `empty`, `cell_name`, `active_mass`, `tot_mass`, `nominal_capacity`, `nom_cap_specifics`, `last_uploaded_from`, `has_no_partial_duplicates` | `summary`, `steps`, `raw`, `cell_name`, `loading`, `meta_common`, `raw_data_files` |
| `batch_tools/batch_core.py` | `data`, `load`, `overwrite_able` | `raw`, `steps` |
| `batch_tools/batch_experiments.py` | `data`, `empty`, `save`, `to_csv`, `make_summary`, `make_step_table`, `set_mass`, `set_nom_cap`, `ensure_step_table` | `summary`, `steps` |
| `batch_tools/batch_analyzers.py` | `data`, `empty` | — |
| `batch_tools/batch_helpers.py` | `get_cycle_numbers`, `get_ccap`, `get_dcap` | `loaded_from` |
| `batch_tools/engines.py` | `data` | `summary` |
| `collectors.py` | `data`, `cell_name`, `cellpy_units`, `raw_units`, `get_cap`, `get_cycle_numbers` | — (units via `c.raw_units`) |
| `easyplot.py` | `data`, `get_cap`, `get_cycle_numbers` | `summary`, `steps` |
| `helpers.py` | `data`, `empty`, `mass`, `drop_from`, `from_raw`, `make_step_table`, `make_summary`, `save`, `to_csv`, `get_converter_to_specific` | `summary`, `steps`, `nom_cap` |
| `ica.py` | `empty`, `get_cap`, `get_ccap`, `get_cycle_numbers`, `load` | — |
| `ocv_rlx.py` | `data`, `get_ocv`, `get_cycle_numbers`, `headers_step_table` | `steps`, `raw` |
| `plotutils.py` | `data`, `cell_name`, `cellpy_units`, `headers_summary`, `raw_units`, `get_cap`, `get_cycle_numbers`, `make_summary`, `unit_scaler_from_raw` | `summary`, `steps`, `raw`, `raw_units`, `nom_cap` |
| `example_data.py` | `save` | — |
| `diagnostics.py`, `live.py`, `processor.py`, `batch_tools/{batch_exporters, batch_reporters, batch_plotters, batch_journals, dumpers, sqlite_from_excel_db}.py` | none directly (operate on DataFrames/journal pages, or go through the experiment objects) | none directly |

---

## 5. Observations

1. **`CellpyCell.data` is the choke point.** Utils rarely hold a `Data` object on its
   own; they always reach it via the `data` property. The effective `Data` contract for
   the utils is small: the three DataFrames (`summary`, `steps`, `raw`), `raw_units`,
   and a handful of meta forwarders (`nom_cap`, `cell_name`, `loading`), plus
   `meta_common`/`raw_data_files`/`loaded_from` in exactly one place each (journal-page
   generation in `batch.py` and `batch_helpers.py`).

2. **The heavily-used `CellpyCell` surface is also small:** `data`, `empty`,
   `cell_name`, `cellpy_units`, `headers_summary`, the capacity getters
   (`get_cap`/`get_ccap`/`get_dcap`/`get_ocv`), `get_cycle_numbers`,
   `make_summary`/`make_step_table`, and IO (`load`/`from_raw`/`save`/`to_csv`).
   Everything else is used in at most one or two places or not at all.

3. **Utils write to `CellpyCell` state directly** in a few spots:
   `ensure_step_table = True` (batch_experiments), `overwrite_able = False`
   (batch_core), `mass` setter (helpers), plus `set_mass`/`set_nom_cap` calls. Any
   refactor should keep these mutation paths (or provide replacements).

4. **Naming collision worth noting:** `ocv_rlx.py` stores a `CellpyCell` in an
   attribute called `self.data` (e.g. `MultiCycleOcvFit`), so code there reads
   `self.data.data.steps` — easy to misread, and easy to break in a refactor.

5. **Dead/duplicated references:** `easyplot.py:726` references `cpobj.steps` (the old
   pre-v8 API) inside a commented-out triple-quoted block — dead code. A few members
   appear only in docstrings (`filtered_summary`, `vacant`, some `get_cap` examples).

6. **Inconsistent access to the same information:** utils mix `c.raw_units`
   (CellpyCell property) with `c.data.raw_units`, and read `nom_cap` via `c.data.nom_cap`
   while `batch.py` reads `cell.nominal_capacity`. Standardizing on one path would make
   the core API boundary cleaner.

---

## 9. Addendum: `filters/` / `exporters/` / `internals/` (2026-07-10)

**Scope:** All `*.py` files under `cellpy/cellpy/filters/`, `cellpy/cellpy/exporters/`,
and `cellpy/cellpy/internals/` (Data/CellpyCell consumers only for the latter).
**Method:** Same AST member-matching rules as §1–§4 above (receiver-aware; excludes
`logging`, `str`, `DataFrame` methods, etc.). Produced with
`.issueflows/00-tools/scan_member_usage.py` and manual spot-check.

### 9.1 Package overview

| Package | Files scanned | `CellpyCell` / `Data` consumer? | Notes |
|---|---|---|---|
| `filters/` | `cycles.py`, `summary.py` | **No** | Generic pandas helpers on caller-supplied frames. |
| `exporters/` | `bdf.py` (+ thin `__init__.py`) | **Yes** (`bdf.py` only) | Thin IO on `cell.data.raw`; delegates cycle filter to `filter_cycles`. |
| `internals/` | `connections.py`, `otherpath.py` | **No** | Path/SSH helpers only; closes G5 negative finding. |

### 9.2 `filters/` — no core-object access

Neither module holds or receives a `CellpyCell` / `Data`. They are **dataframe utilities**
intended for reuse by exporters, plotters, and batch tools:

- `filters/cycles.py` — `filter_cycles(df, …)`; default cycle column from
  `get_headers_normal().cycle_index_txt`.
- `filters/summary.py` — `filter_summary(df, …)`; default `rate_columns` are summary
  header *names* passed in by the caller (see hardcoded-headers addendum).

**Migration implication:** wave-1 port is mechanical (polars expressions + native header
defaults); no utils-contract surface beyond “accept a frame + column names”.

### 9.3 `exporters/bdf.py` — `CellpyCell` members used

Production code (`to_bdf`, `_resolve_filename`, `_build_bdf_frame`):

| Member | Used in | Notes |
|---|---|---|
| `cell_name` | `bdf.py:340` | Default output filename when `filename` is omitted. |
| `headers_normal` | `bdf.py:436` | Resolves raw column names via `_COLUMN_MAP[].cellpy_field` attributes. |
| `data` | `bdf.py:431`, `437` | Entry to raw frame and `raw_units`. |
| `data.raw` | `bdf.py:431` | Source time-series; cycle filter + BDF column build. |
| `data.raw_units` | `bdf.py:437` | pint conversion **source** units (not `cellpy_units`). |

**Not used in production:** `steps`, `summary`, capacity getters, `make_summary` /
`make_step_table`, IO besides export, `cellpy_units` (mentioned in docstring only —
export reads loader units from `data.raw_units`).

**Dependency on `filters/`:** `filter_cycles(raw, …, column=headers.cycle_index_txt)` at
`bdf.py:443`.

**`if __name__ == "__main__"` block (`bdf.py:600–653`):** local debug script only;
repeats `c.data.raw` / `c.data.raw_units` / `c.to_bdf` for manual file checks — not part
of the public export path.

### 9.4 `internals/` — no consumers

`connections.py` and `otherpath.py` implement remote-path handling (`OtherPath`, SSH
checks). No `.data`, `CellpyCell`, or table column access. **No migration work** under
the utils/core contract.

### 9.5 Observations (addendum)

1. **Exporters are a thin slice of the utils contract** — only `data.raw`, `data.raw_units`,
   `headers_normal`, and `cell_name` matter for BDF export today.
2. **Filters stay decoupled** — good for cellpy 2: callers pass native-named frames and
   column parameters; no hidden `CellpyCell` state.
3. **`cellpy_units` vs `raw_units` in BDF** — export correctly uses `data.raw_units`;
   document this distinction when porting (unit plan / `units_label` work).

