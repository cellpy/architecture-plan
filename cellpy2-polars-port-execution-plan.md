# Plan: pandas → polars execution in cellpy 2 (G3)

**Date:** 2026-07-09
**Closes:** gap-analysis item **G3** — turns the findings of the
[pandas-to-polars index-usage report](pandas-to-polars-index-usage-report.md) into an
execution sequence. Core-engine polars work is owned by cellpy-core
(`step-table-polars-migration.md`, issue #13); this plan covers the **cellpy side**.

---

## 1. Decisions

1. **Polars inside, pandas at the edges.** `Data.raw/steps/summary` become polars.
   pandas remains only at explicit boundaries: plotting (matplotlib/plotly get
   `.to_pandas()` at the call site), Excel/journal IO, and the legacy v8 file path
   (pytables is pandas-native — conversion lives inside `cellpy_file/`, nowhere
   else).
2. **No narwhals in the app layer.** cellpy 2 targets polars directly; narwhals is a
   core-internal question (core already decided its stack). Revisit only if a
   downstream consumer demands pandas-frame outputs from the app API — the
   `.to_pandas()` boundary is the answer until then.
3. **The global rule from the report's TL;DR becomes law:** *keys live in columns,
   never in an index.* `datapoint_num` is a column of raw, `cycle_num` a column of
   summary, `filename`/cell-id a column of journal pages.
4. **One flag-day with the header flip.** Frames change name *and* type in the
   native-headers plan's Phase 3 — one migration for consumers, not two. Everything
   in this plan before that day is preparation that keeps v1.x green.

## 2. Phases

### Phase A — de-index in place (v1.x-safe preparation)

Dissolve the three contract-level conventions (report §1) while still on pandas —
every one is redundant state (`drop=False` everywhere), so this is
behavior-preserving:

- A1 `raw` no longer indexed by `data_point`: loaders stop promoting it
  (rides with the [loader plan](cellpy2-loader-port-and-extraction-plan.md) Step 1);
  the defensive restore/check sites in cellreader become asserts, then vanish.
- A2 `summary` no longer indexed by `cycle_index` (`cycle_index_as_index` defaults
  off, deprecated); consumers in helpers/plotutils read the column.
- A3 journal `pages` keyed by a `filename`/cell-id **column**; `.loc[cell_id, …]`
  → mask/join (rides with utils wave 1).

### Phase B — mechanical-pattern cookbook (applied per module, as touched)

The report's §2 catalog becomes the porting cookbook; the recurring replacements:

| Pattern (report §) | Polars replacement |
|---|---|
| positional first/last (2.1) | `.head(1)/.tail(1)`, `pl.first()/pl.last()` over sorted frames |
| masked in-place assignment (2.2) | `with_columns(pl.when(mask).then(...).otherwise(...))` |
| row loops mixing label/positional (2.3) | window expressions / `group_by` aggregations |
| index-aligned concat/Series math (2.4) | explicit `join` on key columns |
| groupby write-back loops (2.5) | `group_by(...).agg(...)` + join back |
| MultiIndex (2.6) | struct columns or flat `key1_key2` columns; wide pivots only at presentation |
| datetime resample (2.8) | `group_by_dynamic` / `upsample` (needs real datetime column) |
| groupby-as-indexed-Series (2.10) | `.agg` + join (delete the `reset_index` cruft) |
| labeled-Series metadata (2.11) | dicts / two-column frames (metadata plan owns) |
| index-shaped masks (2.12) | boolean expressions; align with the engine `mask` column |

### Phase C — the flip (with native-headers Phase 3)

- `Data` frame types change to `pl.DataFrame`; `cellpy_file/` v8 read/write does
  pandas↔polars conversion internally; v9 is parquet-native (no conversion at all).
- The bridge's `pl.from_pandas`/`.to_pandas()` ping-pong disappears together with the
  bridge itself.
- Journal `pages` may lag one wave (stays pandas until utils wave 1 lands) — the
  boundary is `batch.pages` and it is documented as such during the window.

### Phase D — enforcement

- CI lint: `\.index\b` / `set_index` / `reset_index` / `iloc` / `iat` grep-ban
  outside the sanctioned boundary modules (`cellpy_file/`, plotting boundary,
  journal IO). The report's §2.14 list of already-clean boundary code seeds the
  allowlist.
- Benchmarks (release plan §4) run before/after Phase C — the flip must show the
  bridge-removal win, not a regression.

## 3. Test plan

- Phase A: full v1.x suite green per sub-step (the point of doing it early).
- Golden value-parity (shared with the header-migration oracle) after Phase C.
- dtype contract: `RawCols.dtype_map()` asserted on loaded frames.
- Boundary test: every public plotting/export entry point accepts polars-backed
  cells (no `AttributeError: 'DataFrame' object has no attribute ...` leaks).

## 4. Risks

| Risk | Mitigation |
|---|---|
| Hidden index reliance not in the report (it classified ~530 sites, but runtime paths differ) | Phase A asserts before deleting; lint keeps new ones out |
| `group_by` ordering differences (polars doesn't sort keys) | `maintain_order=True`/explicit sort at each converted site (report §2.10 caveat) |
| pytables/v8 path performance with double conversion | conversion confined to `cellpy_file/`; v9 removes it; benchmark guard |
| Plot libs' polars support drifting | keep the explicit `.to_pandas()` boundary; don't rely on implicit dataframe-interchange |

## 5. Effort

Phase A ≈ 2–3 days (mostly deletions + tests) · Phase B is absorbed into
loader/utils waves (cookbook, not a work package) · Phase C ≈ rides the
header-flip PR (+1–2 days for `cellpy_file` conversion internals) · Phase D ≈ 0.5 day.
