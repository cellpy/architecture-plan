# Hard-coded column header names in cellpy

**Date:** 2026-07-08
**Scope:** All `*.py` files under `cellpy/cellpy` (whole package, tests excluded).
**Canonical source of truth:** `cellpy/parameters/internal_settings.py` —
`HeadersNormal` (raw table), `HeadersSummary`, `HeadersStepTable`, `HeadersJournal`,
instantiated as `headers_normal`, `headers_summary`, `headers_step_table`,
`headers_journal` and reachable via `get_headers_normal()` etc.

## The pattern

```python
# BAD — hard-coded column name:
v = cyc_df["voltage"]
ocv_steps = steps.loc[steps["cycle"].isin(cycles), :]

# GOOD — canonical lookup:
from cellpy.parameters.internal_settings import get_headers_normal, get_headers_step_table
hdr_raw = get_headers_normal()
hdr_steps = get_headers_step_table()

v = cyc_df[hdr_raw.voltage_txt]
ocv_steps = steps.loc[steps[hdr_steps.cycle].isin(cycles), :]
```

**Method:** AST scan for string literals equal to any canonical header value, kept only
when appearing in column-access contexts (`df["…"]`, `.loc[…, "…"]`, `groupby`/
`sort_values`/`set_index`/`melt`/`pivot` args, `x=`/`y=`/`color=`/`subset=` kwargs,
rename dicts). 708 raw matches were then classified by receiver/context:

| Category | Count | Verdict |
|---|---|---|
| True hard-coded column names (details below) | ~250 | **Fix** |
| Cellpy-generated curve frames without canonical headers (§5) | ~125 | **Fix differently** — names need a home first |
| String-keyed lookups into headers objects, e.g. `hdr_summary["charge_capacity"]` | 131 | Semi-OK (§6) |
| Unit-dictionary keys (`raw_units = {"current": "A", …}` — `CellpyUnits` keys, not columns) | 163 | Not a finding |
| `kwargs.get/pop("nom_cap")` etc. — function-argument names | 18 | Not a finding |
| Step-type *values* (`steps.type == "charge"`) — cell values, not headers | 12 | Separate issue (§7) |
| Misc. false positives (`log.py` "filename" in logging config, `libs/local_fastnda` vendored schema, mode strings like `cap_type == "charge_capacity"`) | ~10 | Not a finding |

---

## 2. Findings in the core readers

### `readers/cellreader.py`

| Line | Code | Table | Replace with |
|---|---|---|---|
| 3197 | `v = df["voltage"]` | raw | `self.headers_normal.voltage_txt` |
| 4697 | `steps.loc[steps["cycle"].isin(cycles), :]` | steps | `self.headers_step_table.cycle` |
| 2947, 2951, 2955 | `"step"/"type"/"cycle" not in step_specs.columns` (`load_step_specifications`) | steps | `headers_step_table.step` / `.type` / `.cycle` |
| 4307, 4467, 4491, 4523 | `{"capacity": …, "voltage": …}`, `c.insert(0, "cycle", …)` — columns of the frames **returned by** `get_cap`/`get_ocv` | *(none exists)* | see §5 |

`readers/data_structures.py` is clean apart from the legacy-units dict (unit keys, not
columns).

### Instrument loaders (`readers/instruments/`)

| File | Lines | Issue | Replace with |
|---|---|---|---|
| `neware_nda.py` | 28–30, 46, 51–54, 81–84 | Rename-dict *values* are hard-coded cellpy names: `"step_index_txt": "step_index"`, `CHARGE_DISCHARGE_CAP_COLUMNS = {"charge": "charge_capacity_mAh", …}` | `headers_normal.step_index_txt`, `headers_normal.charge_capacity_txt`, … |
| `arbin_sql_csv.py` / `arbin_sql_xlsx.py` | 75/77, 83/85 | Post-processing rename maps vendor name → hard-coded `"power"`, `"dv_dt"` | `headers_normal.power_txt`, `headers_normal.dv_dt_txt` |
| `configurations/maccor_txt_one.py`, `maccor_txt_zero.py` | 65/67, 61/63 | Same pattern: `f"Power(…)": "power"`, `…: "dv_dt"` | same as above |
| `processors/post_processors.py` | 234, 240 | `data.raw.groupby("cycle_index")`, `cycle.groupby("step_index")` | `headers_normal.cycle_index_txt`, `headers_normal.step_index_txt` |
| `maccor_txt.py` (162–474, ~40 sites), `custom.py` (290–293), `arbin_sql.py`:522, `arbin_sql_7.py`:556 | dev/debug plot helpers: `raw.plot(x="test_time", y="voltage")` etc. | Low priority (debug-only code) — `headers_normal.test_time_txt`, `.voltage_txt`, `.current_txt`, `.cycle_index_txt`, `.step_index_txt`, `.charge_capacity_txt`, `.discharge_capacity_txt`, `.data_point_txt` |

`pec_csv.py` (39 matches) is a special case: its translation tables key on cellpy-like
semantic names whose *values* are headers-attribute names (`"voltage": "voltage_txt"`),
which are resolved through `headers_normal` later. By-design, but fragile — the keys
duplicate the canonical names manually.

---

## 3. Findings in utils operating on cellpy tables (raw / steps / summary)

### `utils/ocv_rlx.py` (~25 sites — the worst offender relative to size)

| Lines | Code | Table | Replace with |
|---|---|---|---|
| 88, 369, 374, 380, 384 | `step_table["cycle"]` | steps | `hdr_steps.cycle` |
| 370, 375, 380, 384, 213, 215 | `step_table["type"]`, `final["type"]` | steps | `hdr_steps.type` |
| 135–136 | `row["cycle"], row["step"]`, `row["type"]` (iterating steps) | steps | `hdr_steps.cycle` / `.step` / `.type` |
| 139 | `dfdata["cycle_index"]`, `dfdata["step_index"]` (`dfdata = cellpydata.data.raw`) | raw | `hdr_raw.cycle_index_txt`, `hdr_raw.step_index_txt` |
| 177–186 | `"step_time"`, `"voltage"` on frames sliced from raw | raw | `hdr_raw.step_time_txt`, `hdr_raw.voltage_txt` |
| 196–201, 424–526, 804 | `"cycle"`, `"step"`, `"type"`, `"ir"` keys of the fit-result frames the module builds itself | own schema | module-level constants (or reuse `hdr_steps`) |

### `utils/plotutils.py` (~40 sites)

| Lines | Code | Table | Replace with |
|---|---|---|---|
| 5371, 5539 | `data["cycle_index"]`, `data[…]["step_index"]` (`data = cell.data.raw`) | raw | `hdr_raw.cycle_index_txt`, `hdr_raw.step_index_txt` |
| 5561–5563 | `data.loc[m, "current"/"voltage"/"test_time"]` | raw | `hdr_raw.current_txt` / `.voltage_txt` / `.test_time_txt` |
| 5504–5505 | `table.loc[m_table, "rate_avr"]`, `…"type"` (steps) | steps | `hdr_steps.rate_avr`, `hdr_steps.type` |
| 313 | `df["group"] == group`, `df["sub_group"]` (journal pages) | journal | `hdr_journal.group`, `hdr_journal.sub_group` |
| 5786–6059 | `"cycle"`, `"voltage"`, `color="cycle"`, `y="voltage"` on capacity-curve frames | *(none exists)* | see §5 |

(plotutils also has 7 string-keyed headers lookups, e.g. `step_hdr["cycle"]` at
5381–5383 — see §6.)

### `utils/easyplot.py` (~30 sites)

| Lines | Code | Table | Replace with |
|---|---|---|---|
| 1874, 1888, 1878–1903, 1893 | `steptable[["step_time_avr", "cycle", "type"]]`, `elem[1]["type"]`, `elem[1]["cycle"]` | steps | `hdr_steps.cycle`, `hdr_steps.type` (note: `"step_time_avr"`, `"charge_avr"`, `"discharge_last"` are *aggregated* step columns without canonical entries) |
| 576–1360 (8×), 850–1424 (~14×) | `df.groupby("cycle")`, `cyc_df["voltage"]` on `get_cap` output | *(none exists)* | see §5 |
| 726–739 | `cpobj.steps`, `steptable[["ir", "cycle", "type"]]` — dead commented-out code | — | delete |

### `utils/ica.py` (~11 sites)

623 `cycles_df.groupby("cycle")`, 491/519 `c_first["capacity"]/["voltage"]`,
632–643 `"voltage"/"cycle"/"dq"` in constructed ICA frames — all on `get_cap` output
or self-built dq/dv frames → §5 territory.

### `utils/batch_tools/batch_experiments.py`

506, 831: `summary_tmp.set_index("cycle_index")` → `hdr_summary.cycle_index`.

### `utils/batch_tools/batch_journals.py`

431: `sb["cycle_index"]` → `hdr_summary.cycle_index` (summary frames).

---

## 4. Findings on journal pages (should use `HeadersJournal`)

### `utils/batch.py` (~25 sites) — journal-page builder & accessors

| Lines | Code | Replace with |
|---|---|---|
| 277–292 | `info_df["filename"] / ["mass"] / ["total_mass"] / ["loading"] / ["nom_cap"] / ["label"] / ["cell_type"] / ["nom_cap_specifics"] / ["raw_file_names"] / ["group"] / ["group_label"] / ["cellpy_file_name"]` (285, 287) | `hdr_journal.filename`, `.mass`, `.total_mass`, `.loading`, `.nom_cap`, `.label`, `.cell_type`, `.nom_cap_specifics`, `.raw_file_names`, `.group`, `.group_label`, `.cellpy_file_name` |
| 295, 303, 1216 | `info_df["cellpy_file_name"]`, `info_df["group"]`, `pages["cellpy_file_name"]` | as above |
| 506–520 | `r_pages["group"]`, `r_pages.groupby("group")` (×5) | `hdr_journal.group` |
| 344 | `subset="data_point"` | `hdr_raw.data_point_txt` |

### `utils/helpers.py` (~17 sites)

741, 1056, 1065 `pages.groupby("group")`; 801–802, 1132–1135, 1140, 1348
`pages.loc[cell_id, "group"/"sub_group"/"group_label"/"label"]`; 1051, 1134, 1139, 1347
`"selected"/"group_label"/"label" in pages.columns` →
`hdr_journal.group`, `.sub_group`, `.group_label`, `.label`, `.selected`.

### `utils/collectors.py` (~15 journal sites)

1516–1521, 1538 `b.pages.loc[n, "group"/"sub_group"/"selected"/"label"]`; 1520
`"selected" in b.pages.columns`; 1164–1167, 2676 `"group"/"sub_group" in cols` →
`hdr_journal.*`. (The rest of collectors' matches are curve-frame columns — §5.)

### `utils/batch_tools/batch_journals.py` (~10 sites)

915–961: `self.pages["selected"]`, `self.pages.loc[…, "selected"]` (×9) →
`hdr_journal.selected` (the same file already uses `hdr_journal.group` on line 926 —
inconsistent within one expression).

### `utils/batch_tools/batch_plotters.py`

899 `df["group"] / df["sub_group"]` → `hdr_journal.group` / `.sub_group`;
959–974 axis-label dict keyed on hard-coded **summary** names
(`"cycle_index"`, `"charge_capacity"`, `"discharge_capacity"`, `"charge_c_rate"`,
`"discharge_c_rate"`, `"coulombic_efficiency"`, `"ir_charge"`, `"ir_discharge"`,
`"group"`, `"sub_group"`) → key the dict with `hdr_summary.*` / `hdr_journal.*`.

---

## 5. Columns that have NO canonical header (recommend adding one)

A large block (~125 matches in easyplot, collectors, plotutils, ica) is *not fixable*
with the existing headers classes: the frames returned by `CellpyCell.get_cap`
(and the collectors built on it) use column names that `cellreader.py` itself
hard-codes when constructing the output:

- `"capacity"`, `"voltage"`, `"cycle"`, `"direction"` (get_cap, cellreader.py:4307,
  4523), `"q"`, `"v"` (`collect_capacity_curves` in data_structures.py), `"dq"` (ica)
- aggregated step columns `"step_time_avr"`, `"charge_avr"`, `"discharge_last"`
  (easyplot on steps built with `add_c_rate`-style aggregations)

**Recommendation:** define these once (e.g. a `HeadersCapacityCurves` dataclass, or
extend `HeadersStepTable`/a curves section in `internal_settings`) — then producer
(`get_cap`) and all consumers reference the same object. Fixing the consumers alone
just moves the hard-coding around. This is directly relevant for the cellpy-core split:
the curve-frame schema is currently an implicit contract between `cellreader` and four
utils modules.

---

## 6. Semi-sanctioned: string-keyed lookups into headers objects (131 sites)

`hdr_summary["charge_capacity"]`, `step_hdr["cycle"]`, `hdr_journal["mass"]` — found in
helpers.py (33), batch_plotters.py (29), engines.py (29), sql_dbreader.py (17),
batch_helpers.py (9), json_dbreader.py (8), plotutils.py (7), easyplot.py (5),
cellreader.py (1).

These **resolve correctly** (the string is a key into the headers object, not a column
name pasted into a DataFrame), so they are not bugs. But the key is still a repeated
magic string — a typo raises `KeyError` at runtime instead of `AttributeError`/linter
warnings, and IDE refactoring can't see them. Attribute access
(`hdr_summary.charge_capacity`) is preferable **except** where the postfix mechanism is
needed (`hdr_summary["charge_capacity_gravimetric"]` composes `charge_capacity` +
postfix and has no attribute equivalent — that is the legitimate use of string keys).

---

## 7. Related but out of scope

- **Step-type values** (`steps.type == "charge"`, `"ocvrlx_down"`, …) are hard-coded
  cell *values*, not headers. There is a `list_of_step_types` in internal_settings, but
  no enum/constants used at comparison sites (cellreader.py:6267, 6290,
  ocv_rlx.py:213/215/370/375/380/384, easyplot.py:1878–1903). Same disease, different
  organ — worth its own pass.
- **Unit keys** (`cellpy_units["mass"]`, `raw_units = {"current": "A"}`) are the
  `CellpyUnits` namespace and behave like the headers string-key case (§6).

---

## 8. Suggested priority

1. **Journal pages** (`batch.py`, `batch_journals.py`, `helpers.py`, `collectors.py`,
   `batch_plotters.py`): mechanical, low-risk, high-count — everything maps 1:1 onto
   `HeadersJournal`, and the files already import `hdr_journal` in most cases.
2. **Steps/raw access in `ocv_rlx.py` and `plotutils.py`**: mechanical 1:1 mapping onto
   `HeadersStepTable`/`HeadersNormal`.
3. **Instrument loaders** (`neware_nda.py`, `arbin_sql_csv/xlsx.py`, maccor configs,
   `post_processors.py`): small count, but these define the raw-table contract — a
   header rename would silently break them.
4. **Add canonical names for the capacity-curve frame** (§5), then migrate
   `cellreader.get_cap`, `easyplot`, `collectors`, `ica`, `plotutils` onto them.
5. Optional/stylistic: convert plain string-keyed headers lookups (§6) to attribute
   access; keep string keys only for postfix composition.
6. Delete the dead block in `easyplot.py:721–739`.

---

## 9. Addendum: `filters/` / `exporters/` / `internals/` (2026-07-10)

**Scope:** `cellpy/cellpy/filters/`, `cellpy/cellpy/exporters/`, `cellpy/cellpy/internals/`.
**Method:** AST scan for canonical header string literals in column-access contexts
(same rules as §1). Tool: `.issueflows/00-tools/scan_hardcoded_headers.py`.

### 9.1 Summary

| Package | Verdict | Count |
|---|---|---|
| `filters/cycles.py` | **Clean** | 0 — default cycle column via `get_headers_normal()`. |
| `filters/summary.py` | **Fix** | 2 — default `rate_columns` tuple. |
| `exporters/bdf.py` (production) | **Clean** | 0 — column names resolved via `headers_normal` + `_COLUMN_MAP`. |
| `exporters/bdf.py` (`__main__` debug) | **Low priority** | 2 — hard-coded raw names in local check script. |
| `internals/` | **Clean** | 0 |

### 9.2 `filters/summary.py`

| Line | Code | Table | Replace with |
|---|---|---|---|
| 148 | `rate_columns=("charge_c_rate", "discharge_c_rate")` | summary | `get_headers_summary().charge_c_rate`, `.discharge_c_rate` (or defer to caller after native-headers flip) |

Call sites may override `rate_columns`; the default is the only baked-in literal.

### 9.3 `exporters/bdf.py`

**Production path (`_build_bdf_frame`):** uses `getattr(headers, spec.cellpy_field)` and
`raw[src_col]` — no hard-coded column strings in the export pipeline.

**`if __name__ == "__main__"` block (lines 622–646):** debug-only literals
`c_cha, c_dch = 'charge_capacity', 'discharge_capacity'` for ad-hoc file comparison.
Same class as instrument-loader debug plots — fix when touching the block, or delete when
the manual test script is removed.

### 9.4 `internals/`

No findings. Path helpers do not touch cellpy table schemas.

### 9.5 Suggested priority (addendum)

1. **`filters/summary.py` default `rate_columns`** — one-line fix alongside wave-1 filters
   port (native `HeadersSummary` defaults).
2. **BDF `__main__` literals** — optional / debug-only.

