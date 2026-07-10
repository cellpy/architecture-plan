# Plan: utils / batch / plotting migration to cellpy 2 (G1 + G5 + G10)

**Date:** 2026-07-09
**Closes:** gap-analysis items **G1** (utils/batch/plotting), **G5** (`filters/`,
`exporters/`, unscanned packages), **G10** (live/incremental), and **F6** (surface
the engine features that already exist).
**Foundation:** the [usage report](data-and-cellpycell-usage-in-cellpy-utils.md)
(what utils consume), the [hardcoded-headers report](hardcoded-column-headers-report.md)
(what must be cleaned before renames), the
[native-headers plan](cellpy2-native-headers-migration-plan.md) Phase 4 (when utils
flip), and the [loader/extraction plan](cellpy2-loader-port-and-extraction-plan.md)
§2.3 (`cellpycore.curves`).

---

## 1. Principles

1. **Utils consume the public contract only.** The usage report §5 shows the real
   surface is small: `data` (frames), `empty`, `cell_name`, units/labels, curve
   getters, `get_cycle_numbers`, `make_summary`/`make_step_table`, IO. Formalize that
   as the documented "utils contract"; anything a util needs beyond it is a smell to
   fix in the API, not a private poke (`ensure_step_table = True`,
   `overwrite_able = False`, and `ocv_rlx`'s `self.data`-holds-a-CellpyCell all die
   here).
2. **One migration per module, after the flip.** Utils migrate onto native headers,
   polars frames, `units_label()`, and `cellpycore.curves` in a single pass per
   module — never "headers now, polars later".
3. **Deprecate loudly, delete on schedule** — per the
   [conventions plan](cellpy2-conventions-plan.md).
4. **Feature menu is explicit (F6).** For each module we decide which *new* engine
   capabilities it surfaces: native-only summary columns (energies, powers,
   durations, per-direction stats), `exclude_step_types` summaries (#54), test-level
   summaries, corrected IR semantics (F4).

## 2. Module triage

| Module | Verdict | Notes |
|---|---|---|
| `utils/batch.py` + `batch_tools/*` | **Port** (wave 1) | The core user surface. Journal `pages` keep `HeadersJournal` names (untouched by the native swap) but move keys-to-columns (polars report §1.3); direct pokes (`ensure_step_table`, `overwrite_able`) become load/save kwargs; `engines`/`dumpers` simplify on polars |
| `utils/helpers.py` | **Port** (wave 1) | Summary manipulation → native summary names; `concatenate_summaries` benefits from `test_id`-keyed frames |
| `utils/plotutils.py` | **Port, slim** (wave 2) | On `units_label()` + native names; the raw/steps plot helpers read per-column units; drop duplicated label-composition blocks |
| `utils/collectors.py` | **Port** (wave 2) | Built on `cellpycore.curves`; its own wide/long reshaping goes polars; `BatchSummaryCollector` gains native-only columns as opt-in |
| `utils/ica.py` | **Port** (wave 3) | dq/dv on top of `curves`; scipy stays app-side; spec the ICA output frame while at it |
| `utils/ocv_rlx.py` | **Port, refactor** (wave 3) | Fitting stays app-side; rename the `self.data`-holds-a-CellpyCell attribute; steps access → native names |
| `utils/easyplot.py` | **Deprecate in 2.0, remove in 2.1** | Old API layer over what plotutils/collectors do; contains dead code (easyplot.py:721–739). Decision needed with maintainer — if kept, it gets a rewrite, not a port |
| `utils/diagnostics.py` | Port (small, wave 1) | |
| `utils/live.py` + `utils/processor.py` | **Rebuild on core incremental summarization** (wave 4, G10) | Core ROADMAP marks incremental engine DONE; these two become the consumer. If nobody claims the use case, delete deliberately instead of shipping rotten |
| `utils/example_data.py` | Keep | Regenerate hosted files as v9 once the format lands |
| `filters/cycles.py`, `filters/summary.py` | **Port** (wave 1, G5) | Mask-based selection → polars boolean expressions (index report §2.12). `cycles.py` already uses `get_headers_normal()`; `summary.py` default `rate_columns` should move to `HeadersSummary` (headers addendum §9.2). |
| `exporters/bdf.py` | **Port** (wave 1, G5) | Sole exporter today. Depends on `cell.data.raw`, `data.raw_units`, `headers_normal`, `cell_name`, and `filter_cycles`. `_COLUMN_MAP` is already header-attribute-based; pint conversion from `raw_units`. See usage addendum §9.3. |
| `utils/batch_tools/sqlite_from_excel_db.py`, db readers | Port with batch (wave 1) | Journal/db feed the metadata resolver (metadata plan Step 6) |

## 3. Waves

**Wave 0 — close the scanning blind spot (G5).** Re-run the usage + hardcoded-header
scans over `filters/`, `exporters/`, `internals/` consumers; append findings to the
two reports. Half a day; prevents surprises in every later wave.
**Done (2026-07-10, issue #435):** addenda in the usage and hardcoded-headers reports;
`triage` rows updated for `exporters/bdf.py` and `filters/*`.

**Wave 1 — batch, helpers, filters, diagnostics** (right after native-headers
Phase 3/4). Mostly mechanical: journal keys-to-columns, native summary names,
polars group-bys. This wave defines and documents the **utils contract**.

**Wave 2 — plotting** (needs unit plan Phase 4 `units_label` + curves module).
plotutils and collectors; visual-output smoke tests (see §5).

**Wave 3 — fitting/analysis** (ica, ocv_rlx) on the curves module.

**Wave 4 — live/incremental** (G10): `live.py`/`processor.py` rebuilt as thin
consumers of the core incremental API, or formally dropped. Also the F6 menu
decisions that need new UI: test-level summaries in batch reports,
`exclude_step_types` exposure on `make_summary` kwargs, native-only column plotting.

## 4. Deprecations introduced here

- `easyplot` module-level `DeprecationWarning` (2.0) → removal (2.1), if the
  deprecate verdict stands.
- Legacy curve-frame column names (`"voltage"`, `"capacity"`, `"cycle"`) accepted by
  plotting helpers via the shim for one release.
- `helpers.make_new_cell` (already warns) and any util re-exporting legacy headers.

## 5. Test plan

- Each wave: the module's existing tests ported to native names first (they are the
  characterization net), then extended.
- **Batch end-to-end golden**: journal → load → summaries → report frames for a
  two-cell batch, value-parity vs v1.x on shared columns (mapped through the header
  mapping).
- Plot smoke tests: figures build without error and axis labels come from
  `units_label` (string assertions, not pixel comparisons); optional notebook-based
  visual checks stay manual.
- `filters`: mask semantics parity on golden cells.

## 6. Risks

| Risk | Mitigation |
|---|---|
| Utils quietly depend on pandas idioms (index alignment) that survive review | Wave 0 lint rule from the polars plan (`.index` ban outside boundary modules) applies to utils too |
| easyplot users surprised by deprecation | Decision explicitly with maintainer; if user base unknown, keep one release longer behind a warning |
| Batch journal Excel round-trips (index-labeled sheets) | Journal IO rewritten dict-shaped per metadata plan Step 4 / polars report §2.11 |
| Wave 2 blocked on unit plan Phase 4 slipping | `units_label` is small; pull it forward into wave 1 if needed |

## 7. Effort

Wave 0 ≈ 0.5 day · wave 1 ≈ 4–6 days · wave 2 ≈ 4–6 days · wave 3 ≈ 3–4 days ·
wave 4 ≈ 2–4 days (or 0.5 if dropped). Total ≈ **3 weeks**, parallelizable per module
within a wave.
