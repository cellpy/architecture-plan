# Plan: migrating cellpy 2 to native cellpy-core column headers

**Date:** 2026-07-09
**Goal:** cellpy 2 frames carry the **native cellpy-core names** (`RawCols` /
`StepCols` / `CycleCols`: `potential`, `cycle_num`, `charge_capacity_mean`, …) at all
times. Legacy `Headers*` names (`voltage`, `cycle_index`, `charge_avr`, …) survive only
at explicit compatibility boundaries — reading old cellpy files, optionally writing
them, and a deprecated accessor shim. This **reverses the direction of today's
bridge**: instead of renaming legacy→native→legacy on every engine call, translation
happens **once, at I/O**.

**Builds on (this folder):**
[legacy-cellpy-core-header-swapping.md](legacy-cellpy-core-header-swapping.md) (how the swap works today),
[hardcoded-column-headers-report.md](hardcoded-column-headers-report.md) (literals that must go first),
[cellpy-file-loading-refactor-plan.md](cellpy-file-loading-refactor-plan.md) (the reader/saver seam this plan extends),
[pandas-to-polars-index-usage-report.md](pandas-to-polars-index-usage-report.md) (keys-in-columns rule),
[data-and-cellpycell-usage-in-cellpy-utils.md](data-and-cellpycell-usage-in-cellpy-utils.md) (who consumes which headers).
**Builds on (cellpy-core):** `cellpycore/legacy/mapping.py` (the total, tested
mapping), `column-headers-review.md` (naming decisions: `potential`, seconds;
SPEED-30 deferral), integration roadmap STEP-09.

---

## 1. End state

```
                     ┌────────────────────────────────────────────┐
   old cellpy file   │              cellpy 2 runtime              │   new cellpy file
   (v8 HDF5,         │                                            │   (v9, parquet-based,
    legacy names)    │   Data.raw / steps / summary               │    NATIVE names)
        │            │   = native names, always                   │        ▲
        ▼            │                                            │        │
  cellpy_file.load ──┼─► to_native() ─► engines run WITHOUT any ──┼─► cellpy_file.save
  (legacy_read)      │   (once)         rename sandwich           │   (native, no translate)
        ▲            │                                            │
        │            │   to_legacy() ◄─ optional legacy export ───┼─► save(format="v8")
   v8 write-compat ◄─┘                                            └────────────────────
```

- **One translation module**, fed entirely by `cellpycore.legacy.mapping` (which is
  already lossless/total with exception sets and round-trip tests — STEP-09).
- **`OldCellpyCellCore` retires**; cellpy 2 uses the native `CellpyCellCore`. The
  per-call rename sandwich (today: 3 renames per `make_step_table`, 5+ per summary)
  disappears.
- **The cellpy-file package is the backwards-compatibility tool**: v8 files remain
  readable forever through `legacy_read + to_native`, and *writable* on demand
  (`save(..., format="v8")`) for users who must round-trip with cellpy 1.x.

## 2. Design decisions

### D1 — Translate at I/O, not per call

Today's bridge design exists to keep cellpy 1.x byte-for-byte stable. cellpy 2 has no
such obligation internally; keeping frames native end-to-end removes conversion cost,
removes pandas↔polars ping-pong at the seam, and makes the hard-coded-literal problem
un-reintroducible (the legacy names simply don't exist on the frames).

### D2 — `cellpy/readers/cellpy_file/` is the compatibility boundary

The file-loading refactor plan already isolates reading/writing into
`format.py / read.py / legacy_read.py / write.py` with a version-dispatch registry.
This plan adds:

- `translate.py` — `to_native(data) -> Data` and `to_legacy(data) -> Data`, thin
  orchestration over `mapping.legacy_to_native_raw/step/summary` (+ the postfix
  expansion for `{col}_{gravimetric|areal|absolute}` that the bridge already
  implements in `add_scaled_summary_columns`, cell_core.py:1086–1092 — lift it into
  the mapping helpers so both users share it).
- v8 readers gain a final `to_native()` step **in cellpy 2 only** (v1.x keeps
  returning legacy frames from the same module — the translate step is a flag/wrapper,
  so the module serves both lines during the transition).
- A v9 `CellpyFileFormat` that stores native names and stamps
  `Cols.__version__` + `CELLPY_FILE_VERSION = 9`.
- `cellpy convert` CLI (already planned for v8-upgrades) gains `--to v9`.

### D3 — Import policy for unmapped legacy columns (the exception sets)

`mapping.py`'s `LEGACY_ONLY_*` sets are *documented* but the importer must decide what
happens to the data in those columns. Policy, per class:

| Legacy-only column(s) | Import policy |
|---|---|
| raw `date_time` | **Convert**, don't carry: derive `epoch_time_utc` (int64 ns UTC) via `cellpycore.timestamps`; keep `date_time` as a passthrough extra column for one release, then drop. Needs an explicit timezone rule for naive legacy datetimes (assume local tz of the machine? assume UTC? → **decision needed**, record on `TestMeta.time_zone`). |
| raw `power`, `charge_energy`, `discharge_energy`, `dv_dt`, `sub_step_index/-time`, AC-impedance, `ref_voltage`, `frequency`, `amplitude` | Passthrough unchanged (native schema tolerates extra columns; `validate_raw_frame` warns, doesn't reject). Native cumulative-energy columns are *not* synthesized from legacy energies — different reset semantics (see core issue #42). |
| steps `info`, `ustep`, `ir_pct_change`, `test` | Passthrough. |
| summary cruft (`shifted_*`, `cumulated_ric*`, `cumulated_coulombic_efficiency`, `ocv_*`, `normalized_*`, `low/high_level`) | **Drop on import and recompute on demand.** These are derived columns; cellpy 2's summary accessor recomputes them from the native summary when a legacy consumer asks (the bridge's `_add_legacy_summary_cruft` becomes a free function in cellpy 2's summary utilities). Carrying them risks stale values that no longer match the (re-)computed base columns. |
| summary specific columns `{col}_{mode}` | Rename via the postfix-expanded mapping (mapped 1:1; not exceptions). |
| `test_id` | Not bridged today (exception on both sides). Importer **must** set `test_id = 0` on raw/steps/summary — native group keys are composite `(test_id, cycle_num, …)` (core issue #41/#48). |

`NATIVE_ONLY_*` columns are simply absent when importing old files — the engines
tolerate that; recompute (`make_step_table` / `make_summary`) is the documented way to
get them.

### D4 — Legacy **export** reuses the bridge machinery, demoted to a converter

`to_legacy()` = the inverse renames + column reorder + `_add_legacy_summary_cruft` +
the `index`-column sort quirk — exactly what `OldCellpyCellCore` does today, minus the
engine invocation. Implementation: extract those steps from `cell_core.py`
(`_native_steps_to_legacy`, `_legacy_summary_column_order`, `_add_legacy_summary_cruft`,
date_time re-merge) into `translate.py` / mapping helpers; the bridge then *calls* the
extracted functions (behavior-preserving for v1.x), and cellpy 2's `save(format="v8")`
calls them without the bridge. The byte-for-byte golden tests keep guarding the
extracted code via the v1.x path.

### D5 — Plain `Cols` now; SPEED-30 later

`column-headers-review.md` recommends a future unit+dtype-carrying, versioned header
object (SPEED-30 / `SuperDuperCols`). Do **not** couple this migration to it: ship on
today's plain `Cols` dataclasses (they are tested and spec-pinned), and treat SPEED-30
as an additive upgrade that slots behind the same `Schema` indirection later. (It is
also the natural resolution of the unit plan's per-column-units idea — see the
[gap analysis](cellpy2-plans-gap-analysis.md).)

### D6 — Accessor shim is attribute-level, additive, and temporary

The mapping is deliberately **value-based** (column-name strings). User code and utils,
however, reference headers by *attribute* (`headers_normal.voltage_txt`,
`hdr_summary.charge_capacity`). The shim therefore needs a small attribute-level
table: legacy attribute name → `Schema` path (e.g. `voltage_txt → schema.raw.potential`,
`hdr_steps.cycle → schema.step.cycle_num`, `hdr_summary["charge_capacity_gravimetric"]`
→ `f"{schema.cycle.charge_capacity}_gravimetric"`). ~60 attributes cover everything the
utils actually use (usage + hardcoded reports). Emits `DeprecationWarning`; lifetime
one minor release. Watch the two legacy attributes sharing one value
(`HeadersSummary.discharge_capacity` vs `discharge_capacity_raw`) — the shim maps both
to `schema.cycle.discharge_capacity` with a warning naming the ambiguity.

## 3. Phased plan

### Phase 0 — Prerequisites (all already planned elsewhere; sequence them)

1. File-loading refactor Steps 0–5 land (behavior-preserving, v1.x)
   — gives us `cellpy_file/` with registry, `LoadResult`, `format.py`.
2. Hard-coded header literals fixed per
   [hardcoded-column-headers-report.md](hardcoded-column-headers-report.md) §8
   priorities 1–3 — after this, renaming a header touches header classes only.
3. Canonical names for the capacity-curve frames (headers report §5) decided —
   otherwise `get_cap` consumers break unmanaged when the underlying tables go native.

### Phase 1 — `translate.py` + mapping extensions (v1.x repo, dormant)

- Add `expand_specific_columns(rename, modes)` and attribute-level
  `LEGACY_ATTR_TO_SCHEMA` table to `cellpycore.legacy.mapping` (tested like the rest:
  totality against the legacy dataclass fields).
- Build `to_native(data)` / `to_legacy(data)` in `cellpy/readers/cellpy_file/translate.py`
  implementing D3/D4. Extract the bridge's output-shaping helpers (D4) and re-point the
  bridge at them.
- **Tests:** v8 golden file → `to_native` → `to_legacy` → byte-equality with the
  original frames (modulo the documented drop-and-recompute summary cruft, which gets
  value-equality via recompute instead); totality test that every column in the golden
  v8 file is classified (mapped / passthrough / converted / dropped) — an unclassified
  column fails.

### Phase 2 — v9 format + convert tool

- `CellpyFileFormat` v9 instance: parquet-based container per the metadata plan Step 4
  (sidecar `meta.json` with `"raw_units"` / `"cellpy_units"` / `"limits"` keys), native
  column names, `Cols.__version__` stamped in file metadata, `test_id` present in all
  three frames.
- Registry entries: v9 reader (no translation), v9 writer.
- `cellpy convert old.h5 new.cellpy --to v9` = `load(accept_old=True)` → `to_native` →
  v9 `save`. This is the user-facing backwards-compatibility story: *convert once,
  then live native*.
- **Decision needed** (shared with metadata/file plans): container layout
  (zip-of-parquet vs single arrow file). This plan is layout-agnostic; it only requires
  "native names + stamped header version".

### Phase 3 — Flip the cellpy 2 runtime to native

- cellpy 2 branch: `CellpyCell.core` becomes the lean `CellpyCellCore`;
  `load()` runs `to_native()` on anything < v9; `make_step_table` / `make_summary`
  call the native engines directly — delete the rename sandwich.
- The **oracle changes character**: byte-for-byte legacy parity is no longer the
  target. New oracle = *value parity through the mapping*: for the golden cells,
  `native_pipeline(raw)` renamed to legacy must numerically equal the frozen legacy
  goldens on all **mapped** columns (identical values, legacy column order not
  required). Keep the v1.x byte oracle alive on the v1.x branch only.
- `data.raw` index convention dies here too: keys live in columns
  (polars report §1.1) — fold into the same PR since every loader/consumer is
  already being touched.

### Phase 4 — Consumer surface

- `c.schema` (the injected core `Schema`) becomes the public header API;
  `c.headers_normal` / `headers_summary` / `headers_step_table` become the D6 shim.
- Utils migrate module-by-module in the order of the usage report's per-module table
  (batch/journal first — journal headers are untouched by this plan; then
  helpers/plotutils summary consumers; easyplot last or deprecated).
- Step-type literals migrate to `cellpycore.config.StepType` / `STEP_TYPES` in the
  same pass (same files, same grep).

### Phase 5 — Retire

- Delete `OldCellpyCellCore` from the cellpy 2 dependency path (it stays in
  cellpy-core for as long as v1.x is supported).
- Shim removal one minor release later; `cellpycore.legacy` then serves only
  `cellpy_file/legacy_read + translate`.

## 4. Test plan

| Layer | Test |
|---|---|
| Mapping extensions | postfix expansion bijective; attr-table totality vs legacy dataclass fields |
| Importer | golden v8 → native: every column classified; `test_id==0` present; `epoch_time_utc` round-trips against `date_time` ± tz rule |
| Round-trip | v8 → native → v8 byte-equality (mapped + passthrough), recompute-equality (cruft) |
| v9 | native → v9 → native identity; header version stamp read back |
| Runtime parity | native pipeline vs frozen legacy goldens, value-equality on mapped columns (the Phase-3 oracle) |
| Shim | every legacy attribute resolves, warns once, and equals the schema value |

## 5. Sequencing with the companion plans

- **After**: file-loading refactor Steps 0–5 (hard dependency), hardcoded-literals
  priorities 1–3, curve-frame header decision.
- **With**: metadata plan Step 4 (v9 container is shared), polars port of `Data`
  frames (Phase 3 is the natural flag-day for both — one migration, not two),
  unit plan Phase 4 (`units_label` should be written against *native* names).
- **Before**: utils rewrite for cellpy 2 (utils should migrate onto native headers
  once, not twice).

## 6. Risks

| Risk | Mitigation |
|---|---|
| Timezone semantics of legacy `date_time` → `epoch_time_utc` | Explicit decision + recorded on `TestMeta.time_zone`; property-based round-trip test; default to "naive = local, warn" if undecided |
| Summary cruft recompute ≠ stored legacy values (old bugs frozen in files, e.g. the legacy IR semantics) | Recompute-equality tested on goldens; where legacy values are *known wrong* (see `summary-extractors.md`), document the intentional difference instead of reproducing it |
| Users pass legacy column names into cellpy 2 APIs (`x="voltage"`) | Boundary helpers accept legacy names via the mapping for one release, warn, translate |
| Duplicate-value legacy attrs (`discharge_capacity_raw`) | Shim maps both, warns with disambiguation |
| v1.x and cellpy 2 sharing `cellpy_file/` while diverging | The translate step is a wrapper, not a fork; CI runs the module's tests from both branches during the transition |
| `aux_*` and unknown vendor columns in old files | Passthrough-with-warning is the default; totality test forces a decision per newly-seen column |

## 7. Effort

| Phase | Size |
|---|---|
| 1 translate + mapping extensions + bridge extraction | 2–3 days |
| 2 v9 format + convert | 2–3 days (excluding the container-format decision) |
| 3 runtime flip + new oracle | 3–5 days (the big one; rides with the polars flip) |
| 4 shim + utils first wave | 2–3 days |
| 5 retirement | ½ day |
