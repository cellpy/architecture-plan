# Pandas → Polars porting review: DataFrame index usage in cellpy

**Date:** 2026-07-08
**Scope:** All `*.py` files under `cellpy/cellpy` (whole package; tests and vendored
`libs/local_fastnda` excluded). Context: preparing cellpy 2 / `cellpy-core`, where
frames should be Polars-compatible. Polars has **no row index and no MultiIndex** —
every index-dependent construct in cellpy must be rewritten to operate on columns.

**Method:** ripgrep sweep for `set_index / reset_index / sort_index / reindex /
MultiIndex / idxmax / .index / .loc[ / .iloc[ / .at[ / .iat[ / unstack / resample /
iterrows` (≈530 raw matches, 42 files), then manual classification of each site by
what the index is actually *used for*. Boolean-mask `.loc[mask, cols]` selection and
Python `list.index()` calls (e.g. `cellreader.py:4533`, `cellreader.py:5349`) are not
findings — they port mechanically.

---

## TL;DR

cellpy has **three contract-level index conventions** that must be dissolved before a
polars port, plus about a dozen recurring mechanical patterns. The good news: in almost
every case the indexed key is *also* kept as a column (`set_index(..., drop=False)`) or
defensively restored (`reset_index` guards), so the index is redundant state. The
recommended global rule for cellpy 2:

> **Keys live in columns, never in an index.** `data_point` is a column of `raw`,
> `cycle_index` is a column of `summary`, `filename`/`cell_id` is a column of journal
> `pages`. Anything that today reads `df.index` should read the key column; anything
> that aligns via index should join on the key column.

---

## 1. Contract-level conventions (architectural, fix first)

### 1.1 `raw` indexed by `data_point`

Every instrument loader promotes `data_point` to index (keeping the column,
`drop=False`), and downstream code defensively checks/restores it:

| Where | What |
|---|---|
| `readers/instruments/arbin_res.py:455`, `615–617` | `set_index(hdr_data_point, drop=False)` after load (615 even does `set_index` + `reset_index` back-to-back to sort) |
| `arbin_sql.py:341`, `arbin_sql_csv.py:206`, `arbin_sql_xlsx.py:214`, `arbin_sql_h5.py:205`, `neware_nda.py:255`, `neware_xlsx.py:271` | same pattern, each behind a local `set_index` flag |
| `readers/instruments/processors/post_processors.py:220–224` | `set_index` exists as a *named post-processor step*, switched on in configuration files (`configurations/*.py`: `"set_index": True` in `batmo_bdf_bdf`, `maccor_txt_*`, `neware_txt_*`) |
| `readers/cellreader.py:1684–1685` | re-asserts the index before HDF5 save |
| `readers/cellreader.py:4869–4871` | `has_data_point_as_index()` — public API that inspects `raw.index.name` |
| `readers/data_structures.py:997–1000` | `identify_last_data_point`: falls back to `raw.index.max()` when the column is missing |
| `readers/cellreader.py:5871–5881` | duplicate detection in `make_summary` via `raw.index.is_unique` / `index.duplicated(keep="first")` |

**Suggestion (per case):** keep `data_point` as an ordinary column and delete the
convention entirely.

- Loaders: drop the `set_index` blocks and flags; sort explicitly instead
  (`df.sort(data_point)`), which is what the index was implicitly providing.
- Delete the `set_index` post-processor and the `"set_index": True` config entries.
- `has_data_point_as_index()` → deprecate; `has_data_point_as_column()`
  (`cellreader.py:4873`) already exists and is the polars-compatible check.
- `identify_last_data_point` → the column branch (`raw[data_point].max()`) becomes the
  only branch; polars: `raw.get_column(data_point).max()`.
- Duplicate handling → operate on the column:
  `raw.unique(subset=data_point, keep="first", maintain_order=True)`; the uniqueness
  check becomes `raw[data_point].is_duplicated().any()`.
- HDF5 save: the cellpy-file format currently persists the index. For cellpy 2 the
  parquet/arrow route stores plain columns; for backward compatibility, do
  `reset_index` only inside the legacy pandas I/O boundary.

### 1.2 `summary` indexed by `cycle_index` (`cycle_index_as_index=True`)

`make_summary` hard-codes `cycle_index_as_index = True` (`cellreader.py:5806`,
`5933–5938`; legacy path `6000`, `6139–6144`). This single decision is the source of
most defensive index-juggling in the package:

| Where | What |
|---|---|
| `cellreader.py:643–644`, `706–707` | `split_many` / `with_cycles`: “In case Cycle_Index has been promoted to index” → `reset_index` guard, then `set_index` again on the result (663, 713) |
| `cellreader.py:6518–6521` | `add_to_summary`: dual path — `map(per_cycle)` if the cycle is a column, `per_cycle.reindex(summary.index)` if it is the index |
| `cellreader.py:6577` | `reset_index()` before handing summary to plotting |
| `utils/plotutils.py:1232`, `1239`, `1314`, `3430–3442`, `3573`, `4977–4999` | reset/set/reset dance around every plotting helper (`if hdr not in summary.columns: reset_index`) |
| `utils/helpers.py:923–926`, `1478–1485` | `set_index(x, inplace=True)` so later code can slice on the index; `filtered_summary` *warns and mutates* if the index name is wrong |
| `utils/helpers.py:454–473` | `remove_outliers/last/first_cycles_from_summary`: `s.index.isin(indexes)`, `s.loc[s.index <= last]`, `s.loc[s.index >= first]` — cycle filtering via index comparison |
| `utils/helpers.py:1509–1516` | `summary.index.difference / intersection(filtered_cycles)` then `summary.loc[filtered_index, :]` |
| `utils/helpers.py:1547–1556` | `summary.loc[norm_cycles, col].mean()` — label lookup by cycle number |
| `utils/batch_tools/batch_experiments.py:506`, `831` | `summary_tmp.set_index("cycle_index", inplace=True)` before concatenating cells |
| `utils/plotutils.py:1243`, `1355`, `1455`, `3442`, `3541`, `5009` | `reset_index(drop=True)` sprinkled to neutralize whatever index arrived |

**Suggestion (per case):** never promote `cycle_index`; make “summary has a
`cycle_index` column” the invariant.

- `make_summary`: delete the `cycle_index_as_index` block. This removes the need for
  every guard above in one stroke.
- Cycle filtering: `summary.filter(pl.col(cycle).is_in(cycles))`,
  `summary.filter(pl.col(cycle) <= last)` etc. — the three helpers in
  `utils/helpers.py:440–473` become one-line filters on the column.
- `filtered_summary` index difference/intersection →
  `summary.filter(~pl.col(cycle).is_in(filtered_cycles))` (or without `~`).
- Label lookup means filter + aggregate:
  `summary.filter(pl.col(cycle).is_in(norm_cycles))[col].mean()`.
- `add_to_summary` keeps only the join path:
  `summary.join(per_cycle, on=cycle, how="left")` — the `.map`/`.reindex` fork
  disappears.
- Concatenating per-cell summaries (batch): add the cell id as a column instead of
  relying on index labels (see 1.3 / 2.6).

### 1.3 Journal `pages` indexed by `filename` / cell id

The batch machinery keys the journal table on the index:

| Where | What |
|---|---|
| `utils/batch_tools/batch_journals.py:311–312`, `410`, `462`, `593`, `633` | `pages.set_index(hdr_journal.filename)` on load/save/paste; `meta.set_index("parameter")` |
| `utils/batch_tools/engines.py:382` | `pages.set_index(hdr_journal["filename"], inplace=True)` |
| `utils/helpers.py:277`, `801–802`, `1048`, `1053`, `1070–1073`, `1132–1140`, `1348` | `pages.loc[cell_id, "group"]`, `list(b_sub.index)`, `pages.loc[selector]` — label lookups per cell |
| `utils/batch.py:1203`, `utils/batch_tools/batch_experiments.py:188`, `334–337`, `700–709`, `1041–1053`, `batch_helpers.py:225`, `239` | `pages.iterrows()` where the yielded label *is* the cell id (also `pages.iloc[0:2]`, `pages.loc[indexes]` for “first few cells” / selected cells) |
| `utils/batch_tools/batch_plotters.py:849`, `batch_helpers.py:339` | `pages.reset_index()` to get the filename back as a column |

**Suggestion (per case):** `filename` (cell id) stays a column.

- Row lookup: `pages.filter(pl.col(filename) == cell_id)` or, for a single row,
  `pages.row(by_predicate=pl.col(filename) == cell_id, named=True)`.
- Iteration: `for row in pages.iter_rows(named=True): row[filename], row["group"], …` —
  this also replaces every `iterrows()` site, where the loop body already treats the
  label as the cell name.
- “First N cells” → `pages.head(2)`; “selected cells” → `filter(pl.col(filename).is_in(indexes))`.
- `meta.set_index("parameter")` (journal metadata) → keep two columns
  (`parameter`, `value`) or convert to a dict at the boundary:
  `dict(meta.iter_rows())`.

---

## 2. Mechanical patterns (per case, with polars replacement)

### 2.1 Positional first/last row access (`.iat` / `.iloc[0]` / `.iloc[-1]` / `.iloc[:-1]`)

| Where | What |
|---|---|
| `cellreader.py:3199–3200`, `3273–3274` | `c.iat[-1]`, `c.iat[0]` (per-cycle capacity end-points) |
| `cellreader.py:4253–4263` | local `last(df)/first(df)` helpers wrapping `df.iat[±1]` in `get_cap` |
| `cellreader.py:3756–3777` | `raw[selection].iloc[0][cap_header]` — first value of a masked selection |
| `cellreader.py:6752–6777` | `t1.raw.iloc[:-1]`, `steps.iloc[-1].point_first`, `summary.iloc[-1][cycle]` (drop last row / read last row when merging partial data) |
| `arbin_sql.py:261–264`, `arbin_sql_7.py:267`, `360`, `arbin_sql_h5.py:132–140`, `226`, `arbin_sql_csv.py:209`, `arbin_sql_xlsx.py:230`, `neware_nda.py:261`, `base.py:612`, `pec_csv.py:268` | metadata scalars: `df[col].iloc[0]` / `.iat[0]` (test id, channel, start time…) |
| `post_processors.py:53–56` | `raw.iloc[-2]/[-1].isna().sum()` then `drop(raw.tail(1).index)` — drop trailing junk row |
| `post_processors.py:182`, `196`, `212` | `raw[hdr].iloc[0]` — subtract first time value |
| `post_processors.py:412`, `434` | `charge.iloc[-1]` unpacked into `(index, value)` — **also consumes the index label itself** |
| `utils/helpers.py:1195` | `s.iloc[:-1]` |

**Suggestion:** polars keeps positional access for scalars — `df[0, col]`,
`df[-1, col]`, `series.first()` / `series.last()`, `df.head(df.height - 1)` /
`df.slice(0, -1)`. Per case:

- Masked-first: `raw.filter(selection)[cap_header].first()` or as an expression
  `pl.col(cap).filter(selection).first()`.
- Time-zero subtraction: `pl.col(t) - pl.col(t).first()` (per step:
  `... .over(step_index)` — removes the whole groupby loop, see 2.5).
- `post_processors.py:412/434`: stop returning the index label — select the key
  column explicitly (`charge.select(pl.col(cap).last(), pl.col(volt).last())`).
- Trailing-junk drop: `raw.slice(0, raw.height - 1)` guarded by a null-count check on
  the last row (`raw.tail(1).null_count()`).

### 2.2 Masked in-place assignment (`df.loc[mask, col] = value`)

| Where | What |
|---|---|
| `cellreader.py:553–566` | `_update_cycle_data`: subtract offsets on a cycle mask |
| `cellreader.py:3758–3779` | `_cap_mod_normal`: reset capacities per cycle selection |
| `cellreader.py:4958–4966` | `v.loc[v[voltage] < limit, "is_at_target"] = 1` |
| `utils/helpers.py:1282–1294` | clip out-of-limit values to NaN |
| `biologics_mpr.py:519`, `532` | `mpr_data.loc[i:, step] += 1`, `loc[group.index, step_time] = t` (see 2.5/2.7) |

**Suggestion:** polars frames are immutable; each becomes
`df = df.with_columns(pl.when(mask).then(new).otherwise(pl.col(col)).alias(col))`.
For the pure-flag case (`is_at_target`) it is simpler still:
`pl.when(cond).then(1).otherwise(0)`. Since cellpy frequently mutates `self.data.raw`
in place, callers must be audited to rebind the frame (`data.raw = data.raw.with_columns(...)`)
— silent aliasing bugs are the main porting risk here.

### 2.3 Row loops over `summary.index` mixing label and positional access

| Where | What |
|---|---|
| `cellreader.py:6375–6403` | `_ir_to_summary`: `for i in summary.index: cycle = summary.iloc[i][cycle_col]` then per-cycle `raw.loc[...].values[0]` |
| `cellreader.py:6428–6439` | `_end_voltage_to_summary`: same shape, takes `.values[-1]` |
| `cellreader.py:3705–3711` | `_cap_mod_summary`: `iterrows()` computing a running difference, written back via `summary.loc[index, col] =` |

Note: `for i in summary.index: summary.iloc[i]` treats index *labels* as *positions* —
already a latent bug the moment the index is not a clean RangeIndex, and one more
reason to remove convention 1.2.

**Suggestion:** replace the loops with aggregations + joins:

- IR / end-voltage: build a per-cycle frame from `raw` in one pass —
  `raw.filter(step-mask).group_by(cycle).agg(pl.col(ir).first())` (or `.last()` for end
  voltage) — then `summary = summary.join(per_cycle, on=cycle, how="left").fill_null(0)`.
  This also removes the `only_zeros + ir_values` index-aligned Series arithmetic at
  `cellreader.py:6400–6403`.
- Running difference: `pl.col(cap).diff().fill_null(pl.col(cap).first())` — the whole
  `iterrows` body is one expression.

### 2.4 Index-aligned horizontal composition (`concat(axis=1)`, Series arithmetic, `reindex`)

| Where | What |
|---|---|
| `cellreader.py:4122`, `4160` | `pd.concat([v, dc], axis=1)` — voltage/capacity columns glued by index |
| `arbin_res.py:636` | `pd.concat(aux_dfs, axis=1)` — wide aux channels |
| `cellreader.py:6400–6403` | `only_zeros + ir_values` then `summary.insert(0, ...)` |
| `cellreader.py:6521` | `per_cycle.reindex(summary.index)` (see 1.2) |
| `cellreader.py:5354`, `utils/helpers.py:162` | `reindex(columns=...)` — column ordering/selection only |
| `utils/collectors.py:699`, `1193` | `sort_index(axis=1)` on MultiIndex columns |

**Suggestion:**

- Same-length columns derived from the same frame: build them in a single
  `select`/`with_columns` instead of concatenating Series
  (`raw.select(pl.col(v), dc_expr.alias(cap))`).
- Anything joining data of *different* lengths (aux channels, per-cycle values into
  summary) must become an explicit `join` on `data_point` / `cycle` — index alignment
  is the one pandas behavior polars deliberately refuses to emulate.
- `reindex(columns=cols)` → `df.select([c for c in cols if c in df.columns])`, adding
  `pl.lit(None).alias(c)` for genuinely missing columns if the old fill-behavior is
  wanted.

### 2.5 Groupby write-back loops (`for name, group in df.groupby(...)` + `.loc[group.index, ...]`)

| Where | What |
|---|---|
| `biologics_mpr.py:527–532` | per-step `step_time = t - t[0]`, written back via `loc[group.index, ...]` |
| `post_processors.py:234–249` | cumulative capacities: nested cycle/step loop, `step.at[step.index[-1], hdr]` reads the last row, frames re-concatenated |
| `cellreader.py:4723–4738` | `for name, group in selected_df.groupby([cycle, step])` interpolating per group, re-`concat` |
| `cellreader.py:3883–3899` | `gb = c.groupby(cycle_index_header)` then per-group first/last |

**Suggestion:** window expressions remove both the loop and the index write-back:

- Step time: `(pl.col(t) - pl.col(t).first()).over(step_index)`.
- Cumulative capacities across steps: shift-last-cumsum per cycle —
  `pl.col(cap) + pl.col(cap).last().over([cycle, step]).shift(...)`, or in the general
  case `group_by([cycle, step], maintain_order=True).agg(...)` + join back.
- Per-group interpolation (genuinely needs a Python callable):
  `df.group_by(keys, maintain_order=True).map_groups(fn)` — the escape hatch exists,
  no index needed.

### 2.6 MultiIndex creation and consumption

| Where | What |
|---|---|
| `utils/collectors.py:1633–1635`, `1716` | `pd.concat(all_curves, keys=keys, names=["cell", "point"]).reset_index(level="cell")` — row MultiIndex used purely to inject a `cell` column |
| `utils/batch_tools/batch_plotters.py:217` | `isinstance(charge_capacity.columns, pd.MultiIndex)` — behavior forks on column MultiIndex |
| `utils/batch_tools/batch_helpers.py` (`join_summaries`) & `batch_experiments.py:506`, `831` | per-cell summaries concatenated into MultiIndex columns |
| `arbin_res.py:646–647` | `set_index([point, aux_index, data_type]).unstack(2).unstack(1)` — long→wide aux data |
| `utils/collectors.py:699`, `1193` | `sort_index(axis=1, level=...)` on the wide result |

**Suggestion:** polars has neither row nor column MultiIndex.

- `concat(keys=...)` → tag before stacking:
  `pl.concat([df.with_columns(pl.lit(name).alias("cell")) for name, df in ...])`.
  This is *less* code than the pandas version and removes the `reset_index(level=...)`.
- Wide multi-level columns → prefer **long/tidy** frames (`cell`, `cycle`, `variable`,
  `value`) as the canonical batch format; where a wide table is really needed
  (plotting), `df.pivot(on="cell", index=cycle, values=var)` produces flat string
  columns. Compound names (`f"{value}_{cell}"`, which polars pivot generates) replace
  the two-level header; the `isinstance(..., MultiIndex)` fork disappears.
- Aux unstack → `aux.pivot(on=[data_type, aux_index], index=data_point)`.

### 2.7 Index-slice arithmetic on flag positions

`biologics_mpr.py:517–519`: collect row labels where a flag fires
(`mpr_data[n].index.values`), then `for i in ns_changes: mpr_data.loc[i:, step] += 1`.

**Suggestion:** this is a cumulative sum in disguise:
`step_index = 1 + pl.col(flag).cast(pl.Int64).cum_sum()` — one expression, no index, no
loop, O(n) instead of O(n·changes).

### 2.8 Datetime-index resampling

`cellreader.py:4951–4954` (`time_at_voltage_level`): `set_index(date_time)` →
`resample(unit).ffill().bfill()`.

**Suggestion:** polars resamples on a **sorted column**, not an index:
`v.sort(date_time).upsample(time_column=date_time, every="1s").fill_null(strategy="forward").fill_null(strategy="backward")`
(or `group_by_dynamic` for down-sampling). The duplicate-timestamp guard at `:4949`
becomes `v.unique(subset=date_time, keep="first")`.

### 2.9 Reverse-order slicing

`cellreader.py:4513–4516`: `_first_df.loc[::-1].reset_index(drop=True)`.
**Suggestion:** `df.reverse()` — direct equivalent, no index involved.

### 2.10 Groupby results used as indexed Series

| Where | What |
|---|---|
| `cellreader.py:5554–5559` | `.groupby(cycle).last().values.ravel()` — index discarded via `.values` |
| `cellreader.py:5618–5619` | `groupby(cycle).sum()` then `reset_index()` to recover the key |
| `cellreader.py:4782–4786`, `2966`, `utils/ica.py:341` | `groupby(...).agg(...).reset_index()` |
| `cellreader.py:6516` | `raw.groupby(cycle)[col].agg(method)` consumed via `.map` / `.reindex` (see 1.2) |

**Suggestion:** free wins — `group_by` in polars *never* produces an index, so every
`.reset_index()` after an aggregation is simply deleted, and `.map(per_cycle)` becomes
a join. Only caveat: add `maintain_order=True` (or an explicit `sort`) where cycle
order matters, since polars `group_by` does not sort by key the way pandas does.

### 2.11 Labeled-index Series as key–value metadata

| Where | What |
|---|---|
| `cellreader.py:3415–3417` | `cellpy_units.index = "cellpy_units_" + cellpy_units.index` — prefixing Series labels for the meta frame |
| `cellreader.py:6068` | `summary.index = list(range(summary_length))` |
| `neware_xlsx.py:351–395` | spreadsheet metadata read by absolute cell position `unit_frame.iloc[2, [2, 6]]`, `set_index("name").to_dict()` |
| `utils/batch_tools/sqlite_from_excel_db.py:75` | `sheet.set_index(COLUMNS_RENAMER["id"])` before `to_sql` |

**Suggestion:** these are dict-shaped data wearing a Series costume. Represent
metadata as plain dicts or two-column (`key`, `value`) frames; the prefixing becomes a
column operation (`pl.format("cellpy_units_{}", pl.col("key"))`). The neware xlsx
positional cell-reads can stay pandas/openpyxl at the ingest boundary (they read a
spreadsheet *layout*, not a table) — polars also supports `df[row, col]` if converted.
`summary.index = range(...)` is deleted outright (row numbers, when truly needed:
`with_row_index()`).

### 2.12 Index-shaped boolean masks

`filters/cycles.py:65`, `filters/summary.py:105`, `122`, `203`:
`mask = pd.Series(True, index=df.index)` combined with `&=`.

**Suggestion:** compose expressions instead of materialized masks:
start from `pl.lit(True)` and `&` on `pl.col(...)` predicates, applying once via
`df.filter(expr)`. Same shape, no index.

### 2.13 `dbreader.py` (simple-db excel reader)

23 hits, nearly all `sheet.loc[:, col]` column slices plus boolean-row selection
(`dbreader.py:385–400`, `628–699`) — no real index dependence except returning
`sheet.loc[rows, :]`. **Suggestion:** `sheet.select(col)` / `sheet.filter(pred)`;
port mechanically.

### 2.14 Boundary code that already strips indexes

`exporters/bdf.py:538`, `589` and `readers/data_structures.py:1344` use
`reset_index(drop=True)` to *remove* index effects before export — these confirm the
index carries no information and can be deleted once frames are index-free.
`cellreader.py:6766` `concat(...).reset_index(drop=True)` → `pl.concat` (vertical) needs
nothing.

---

## 3. Note on the cellpy-core seam

`make_step_table` and `make_summary` already delegate to
`cellpycore` (`cellreader.py:3075`, `5908–5921`), and the seam comments describe the
core as doing “cycle-end selection + **index reset** + column pruning”. A quick sweep
of `cellpy-core/src/cellpycore` finds ~100 index-related hits (`cell_core.py`,
`summarizers.py`, `merge.py`, …) — i.e. the pandas index conventions have leaked across
the seam. Recommendation: adopt the “keys are columns” contract in `cellpycore`
*first* (it is the code being rewritten in polars), then make cellpy 1.x conform at the
boundary (`reset_index` on entry, nothing on exit). A follow-up review scoped to
`cellpycore` is worth doing before the polars swap.

---

## 4. Suggested porting rules (summary)

| # | Pandas idiom in cellpy | Polars replacement |
|---|---|---|
| 1 | `set_index(key, drop=False)` conventions (raw/`data_point`, summary/`cycle_index`, pages/`filename`) | delete; key stays a column; sort explicitly where order mattered |
| 2 | Label lookup `df.loc[label, col]` | `df.filter(pl.col(key) == label)` + `.item()` / `row(by_predicate=...)` |
| 3 | Positional scalars `.iat/.iloc[0/-1]` | `df[0, col]`, `.first()/.last()`, `head/tail/slice` |
| 4 | `df.loc[mask, col] = value` | `with_columns(pl.when(mask).then(...).otherwise(pl.col(col)))` + rebind |
| 5 | Index alignment (`concat(axis=1)`, `reindex`, Series arithmetic) | explicit `join` on key columns; single `select` for same-frame columns |
| 6 | `iterrows` / `for i in df.index` loops | `group_by(...).agg`, window `.over(...)`, `diff/cum_sum`; `iter_rows(named=True)` only at orchestration boundaries (batch pages) |
| 7 | MultiIndex rows (`concat(keys=...)`) | tag with `pl.lit(name).alias("cell")` before vertical concat |
| 8 | MultiIndex columns / `unstack` | long/tidy format as canonical; `pivot` with flat compound names for display |
| 9 | `resample` on DatetimeIndex | `upsample`/`group_by_dynamic` on sorted datetime column |
| 10 | `.index.duplicated / is_unique / max` | same operations on the key column (`unique(subset=...)`, `is_duplicated`, `max`) |
| 11 | `groupby(...).agg(...).reset_index()` | `group_by(..., maintain_order=True).agg(...)` — reset_index deleted |
| 12 | Labeled Series as key–value store | dict or (`key`,`value`) frame |

**Priority order:** 1.2 (summary index — largest blast radius, removes dozens of
defensive guards), then 1.1 (raw index — touches every loader plus file I/O), then 1.3
(batch/journal), then the mechanical patterns file by file. The row-loop sites (2.3,
2.5, 2.7) deserve early attention regardless of the port: they are also the known
performance hot spots (`cellreader.py:5551` TODO says as much) and one
(`for i in summary.index` + `iloc[i]`) is a latent correctness bug today.
