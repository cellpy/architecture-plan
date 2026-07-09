# Plan: metadata handling in cellpy 2 (common + test-dependent)

**Date:** 2026-07-08
**Scope:** How cellpy 2 should represent, populate, access, persist, and merge
cell-level ("common") and test-level ("test-dependent") metadata.
**Builds on (this folder):**
[data-and-cellpycell-usage-in-cellpy-utils.md](data-and-cellpycell-usage-in-cellpy-utils.md) (§2.2, §5 — who consumes meta),
[pandas-to-polars-index-usage-report.md](pandas-to-polars-index-usage-report.md) (§2.11 — meta stored as index-labeled frames),
[legacy-cellpy-core-header-swapping.md](legacy-cellpy-core-header-swapping.md) (the mapping/totality pattern to reuse),
[hardcoded-column-headers-report.md](hardcoded-column-headers-report.md) (canonical-names discipline).
**Builds on (cellpy-core design docs):**
`.issueflows/04-designs-and-guides/metadata-scaffolding.md`,
`test-metadata-and-merging.md`, `cellpy-core-migration.md` §4.

---

## 1. Where we are

### 1.1 Legacy cellpy (v1.x)

- Two dataclasses in `cellpy/parameters/internal_settings.py`:
  `CellpyMetaCommon` (~28 fields: cell/material/geometry **plus** test-ish fields like
  `cell_name`, `start_datetime`, `time_zone`, `tester_ID`, `raw_id`,
  `cellpy_file_version`) and `CellpyMetaIndividualTest` (8 fields: `channel_index`,
  `creator`, `schedule_file_name`, `test_type`, `voltage_lim_low/high`, `cycle_mode`,
  `test_ID`). One **scalar instance of each per `Data`** object
  (`data_structures.py:561–563`).
- Access is spread over **three paths**: `c.mass` (CellpyCell property) →
  `c.data.mass` (Data property forwarder, ~15 of them at
  `data_structures.py:579–664`) → `c.data.meta_common.mass` (the actual field).
  Utils mix all three (usage report §5.6).
- **Config defaults are baked into the class definition** (`prms.Materials.default_mass`
  etc. as dataclass defaults), so schema and site configuration are coupled at import
  time.
- Population protocols are dict-shaped and HDF5-flavored: `update()` / `digest()`
  consume `{key: [values]}` lists (`internal_settings.py:80–120`); loaders write fields
  directly (`data.start_datetime = ...`, `arbin_sql_h5.py:132–140`).
- Persistence: everything is flattened into **one pandas "infotable"** for HDF5
  (`cellreader.py:2445–2460`), with `raw_units_*` / `raw_limits_*` **crammed into the
  same table via string key prefixes**, and read back by prefix-popping
  (`cellreader.py:2351–2377`). Excel export builds index-labeled frames and prefixes
  the *index* (`cellreader.py:3412–3420`) — flagged as polars-incompatible in the index
  report §2.11.
- The utils' actual meta consumption is tiny (usage report §2.2, §5.1): `nom_cap`,
  `cell_name`, `loading`, `mass`/`tot_mass` setters, and the whole `meta_common` object
  in exactly one place — journal-page generation (`batch.py:283`).
  `meta_test_dependent` is consumed by **no** util.

### 1.2 cellpy-core (the cellpy 2 engine)

Scaffolding already exists and encodes the decisions we should follow
(`cellpycore/metadata/`):

- `CellMeta` — cell-dependent, ~19 optional fields (mined from `CellpyMetaCommon`,
  cell/material/geometry only).
- `TestMeta` — test-dependent, **one record per `test_id`** (the compact key that also
  lives on every harmonized-raw row), including provenance
  (`source_kind/type/uri/uuid`, `raw_file_names`, `loaded_datetime`) and the fields
  that legacy misfiled under "common": `cell_name`, `start_datetime`, `time_zone`,
  `tester_id`. Optional `cell: CellMeta` link.
- `TestMetaCollection` — `dict[int, TestMeta]`, the "many merged test files" container.
- `io.py` — real stdlib `to_dict/from_dict/to_json/from_json` + `merge_test_meta`
  (renumbering on `test_id` collisions); **deliberate stubs** for HDF5 archive and
  DB/API (JSON-LD) transport.
- Boundary rule (`cellpy-core-migration.md` §4): **core owns shape and tools; cellpy 2
  owns content, population policy, and persistence.** Engines must keep working with
  empty metadata.

### 1.3 The gaps this plan must close

| # | Gap | Evidence |
|---|---|---|
| G1 | Scalar `meta_test_dependent` cannot hold N merged tests | `test-metadata-and-merging.md`; `TODO: v2.0 edit this from scalar to list` |
| G2 | Legacy common/test split is wrong — test fields live in `CellpyMetaCommon` | compare `internal_settings.py:137–203` with core `TestMeta` |
| G3 | Units/limits mixed into the meta table via string prefixes | `cellreader.py:2351–2377`, `2452–2460` |
| G4 | Meta serialized as index-labeled pandas frames | index report §2.11 (`cellreader.py:3412–3420`, `to_frame()` at `internal_settings.py:122–133`) |
| G5 | Three access paths for the same value | usage report §5.6 |
| G6 | Config defaults baked into schema | `CellpyMetaCommon` field defaults |
| G7 | `TestMetaCollection` not wired onto `Data`; no `test_id` on steps/summary | `metadata-scaffolding.md` "Deferred"; `test-metadata-and-merging.md` "not yet implemented" |
| G8 | No real persistence (HDF5 stubs; parquet story undefined) | `io.py:123–192` |
| G9 | Legacy fields with no core home (`tester_server/client_software_version`, `tester_calibration_date`, `file_errors`, `raw_id`, `cellpy_file_version`, `channel_index` naming) | field diff §3.1 below |

---

## 2. Target model

```
CellpyCell (cellpy 2)
 └── Data
      ├── raw / steps / summary        polars frames, each carrying test_id
      ├── meta:  CellMeta              ← "common meta" (one physical cell)
      ├── tests: TestMetaCollection    ← "test-dependent meta" {test_id: TestMeta}
      ├── raw_units / cellpy_units     separate structures — never inside meta
      └── raw_limits                   separate structure — never inside meta
```

Principles (each fixes a numbered gap):

1. **One schema, owned by core** (`cellpycore.metadata.models`). cellpy 2 does not
   define its own meta dataclasses; it imports them. Legacy names survive only in the
   migration mapping (G2).
2. **Common meta = `CellMeta`, test-dependent meta = `TestMetaCollection`.** The
   single-test case is `tests[0]` — same model, no special casing (G1, G7).
3. **One canonical access path:** `c.data.meta.<field>` and
   `c.data.tests[tid].<field>`. Keep a *thin, deprecated* property facade for the
   handful of names the utils actually use (`mass`, `tot_mass`, `nom_cap`,
   `cell_name`, `loading`, `start_datetime`) during the transition, emitting
   `DeprecationWarning` (G5). The usage report shows this facade only needs ~8 names.
4. **Convenience accessors for the single-test case:** `Data.active_test` property
   (returns `tests[0]` when exactly one test, raises/None otherwise) so
   `c.data.active_test.cycle_mode` reads naturally; `c.data.cycle_mode` goes through
   it during deprecation.
5. **Units and limits are not metadata.** They keep their own containers and their own
   serialization blocks; the prefix-key hack dies with the infotable (G3).
6. **Defaults are applied by a resolver, not the schema** (G6): a bare `CellMeta()` is
   all-`None`; population happens explicitly with documented precedence (§4).
7. **Serialization is dict/JSON-shaped, never an indexed frame** (G4): `to_dict`/
   `to_json` from `cellpycore.metadata.io` everywhere; tabular *views* (Excel sheets,
   journal pages) are built as plain two-column (`key`, `value`) or per-test-row
   frames at the boundary.

---

## 3. Implementation plan

### Step 1 — Field mapping legacy ⇄ core (the "meta mapping" module)

Reuse the pattern that already works for column headers
(`legacy-cellpy-core-header-swapping.md`): declare the translation **once**, as
explicit pair tables + exception sets, with totality tests that fail when a field is
added on either side without being categorized.

Deliverable: `cellpycore/legacy/meta_mapping.py` with:

- `COMMON_TO_CELL_PAIRS` — `CellpyMetaCommon` field → `CellMeta` field (mostly
  identity: `material`, `mass`, `tot_mass`, `nom_cap`, `nom_cap_specifics`,
  electrode/electrolyte/separator fields, `comment`).
- `COMMON_TO_TEST_PAIRS` — the **re-homed** fields: `cell_name`, `start_datetime`,
  `time_zone`, `tester_ID → tester_id`.
- `INDIVIDUAL_TO_TEST_PAIRS` — `channel_index → channel`, `creator`,
  `schedule_file_name`, `test_type`, `voltage_lim_low/high`, `cycle_mode`,
  `test_ID → test_id` (with int coercion; legacy sometimes holds lists — take first,
  log the rest).
- `LEGACY_ONLY` — fields that die or move to other layers, each with a documented
  destination: `cellpy_file_version` → file-format layer; `raw_id` → superseded by
  `TestMeta.uuid`; `file_errors` → drop (marked "not in use" in legacy);
  `tester_server_software_version`, `tester_client_software_version`,
  `tester_calibration_date` → see G9 decision below.
- `CORE_ONLY` — fields legacy never had (`test_family`, `source_*`, `uuid`,
  `loaded_datetime`, …) that migration fills from context: `source_kind=FILE`,
  `source_uri=loaded_from`, `raw_file_names` from `raw_data_files`,
  `loaded_datetime=now`.

**G9 decision needed:** either add the three tester-calibration/software fields to
`TestMeta` (they are legitimately test-dependent provenance; my recommendation), or
add a spec'd `extra: dict[str, str]` escape field on both models. Do **not** silently
drop populated values.

**Also decide here** *(added 2026-07-09, cross-check vs unit-handling-cellpy2-plan.md)*:
`CellMeta.volume`. The unit plan's Phase 5 implements volumetric specific-capacity
mode (`nominal_capacity_as_absolute` currently raises `NotImplementedError` for it),
which needs a volume on the cell; legacy carries it only as commented-out property
stubs (`data_structures.py:599–605`). Recommendation: add `volume: Optional[float]`
to `CellMeta` alongside `active_electrode_area` (cellpy units convention, `cm**3`).

Tests: bijectivity of pairs, round-trip identity, totality
(fields(legacy) == mapped ∪ LEGACY_ONLY, fields(core) == mapped ∪ CORE_ONLY) —
mirroring `test_header_mapping.py`.

### Step 2 — Wire the models onto `Data` (core, then cellpy 2)

- Replace the scalar `meta_test_dependent` mock on core's `Data` with
  `tests: TestMetaCollection` and add `meta: CellMeta` (defaulting to empty instances —
  the engines already must tolerate empty meta).
- Add `test_id` to `StepCols` / `CycleCols` and switch engine group keys to composite
  `(test_id, cycle_num, …)` as specified in `test-metadata-and-merging.md`. This is
  the prerequisite for meaningful multi-test summaries and is tracked with the engine
  work — coordinate, don't duplicate.
- `CellpyCellCore.cycle_mode` reads via `active_test` (kills the flagged TODO).

### Step 3 — Population pipeline in cellpy 2 (loaders + resolver)

- **Loader contract:** each instrument loader returns a *draft* `TestMeta`
  (+ optionally a partial `CellMeta` when the file carries masses etc., e.g.
  `arbin_sql_h5.py:139–140`, `neware_xlsx.py:393–394`) instead of poking attributes
  onto `Data`. The loader fills only what the file actually knows; provenance fields
  (`source_kind/type/uri`, `raw_file_names`) are filled by the loading framework, not
  by each loader.
- **`MetaResolver`** (new, small): merges the layers into the final objects with fixed
  precedence —

  `explicit user kwargs  >  journal/db values  >  raw-file values  >  prms config defaults`

  This is where `prms.Materials.default_mass` etc. move to (G6). The resolver logs
  which layer won for each populated field (cheap provenance: a `dict[field, layer]`
  kept next to the meta, useful for debugging batch runs; can be dropped if it proves
  noisy).
- The dict-shaped `update()`/`digest()` protocols are retired; `from_dict` (io.py)
  covers the legitimate use.

### Step 4 — Persistence in cellpy 2

The new cellpy-file is parquet/arrow-based (per the polars port), so:

- **Canonical form:** one JSON document per level —
  `{"cell": to_dict(meta), "tests": {tid: to_dict(t)}, "raw_units": ...,
  "cellpy_units": ..., "limits": ...}` — stored as (a) a sidecar `meta.json` inside
  the cellpy-file container (zip/directory), or (b) arrow schema/file metadata on the
  raw table if a single-file format is chosen. Either way the unit/limit blocks are
  separate keys, not prefixed rows (G3, G4). *(Amended 2026-07-09, cross-check vs
  unit-handling-cellpy2-plan.md: split the single `"units"` key into explicit
  `"raw_units"` and `"cellpy_units"` — the unit plan's Phase 5 requires both to
  round-trip; today only raw_units travels with the data.)*
- Implement `load_archive`/`save_archive` **in cellpy 2** against this format; the
  core stubs stay stubs (boundary rule).
- **Legacy import shim** (read-only): v8/v9 HDF5 infotable → `meta_mapping` →
  `CellMeta` + `TestMeta(test_id=0)`; reuse the existing prefix-popping only inside
  this shim. The pre-v8 upgrade chain (`_extract_meta_from_old_cellpy_file_max_v7`)
  is ported as-is into the shim module and never touches the new models directly —
  it produces a legacy dict that then goes through the same mapping.
- Excel/report export: build the meta sheets from `to_dict()` as two-column frames
  (`key`, `value`) plus one row-per-test frame for `tests` — no index manipulation.

### Step 5 — Merging

- Raw merge = vertical concat + `test_id` assignment (already designed); metadata
  merge = `merge_test_meta` (already implemented, including renumbering) — cellpy 2's
  `merge()` calls it and *also renumbers the `test_id` column in the frames
  consistently* (the collection and the frames must renumber together; add a test).
- **`CellMeta` conflict policy** (new, needed): merging tests of the *same* cell —
  field-wise: equal or one-side-`None` → keep value; true conflict → keep first,
  `warnings.warn` with both values. Merging *different* cells is an error unless
  forced.
- Legacy `merge` reads `meta_common.start_datetime` for ordering
  (`cellreader.py:2598–2599`) — in cellpy 2 this becomes per-test
  `tests[tid].start_datetime`, which is more correct anyway.

### Step 6 — Consumer migration (batch/journal/utils)

- Journal-page generation (`batch.py:267–289`) is the only util that consumes
  `meta_common` wholesale — port it to `to_dict(c.data.meta) | to_dict(active_test)`
  and pick the journal columns from that dict.
- The setters utils use (`set_mass`, `set_nom_cap`, `c.mass = ...`,
  batch_experiments `set_mass/set_nom_cap` calls) route to `data.meta` through the
  facade; keep them working.
- `dbreader`/simple-db and the journal remain *sources* feeding the resolver
  (journal/db layer), not alternative stores.

### Step 7 — DB/API (JSON-LD) — deferred, but keep the door open

`fetch_from_db`/`push_to_db` stay stubs until there is a concrete consumer. The only
action now: ensure `TestMeta.uuid` is **assigned at first load** (uuid4) and preserved
through save/load/merge, so records are linkable later. Add that to Step 3.

---

## 4. Test plan (definition of done per step)

| Step | Tests |
|---|---|
| 1 | mapping bijectivity / round-trip / totality (add-a-field breaks the build) |
| 2 | engines run with empty meta (existing guard); composite-key group-bys on a two-test merged frame |
| 3 | resolver precedence matrix (4 layers × override/absent); loader drafts contain only file-known fields |
| 4 | save→load round-trip equality for `CellMeta` + `TestMetaCollection` (incl. `uuid`, enum coercion); v9 HDF5 fixture imports to expected models; v5 fixture through the upgrade chain |
| 5 | merge renumbers collection *and* frames consistently; CellMeta conflict warning fires |
| 6 | journal pages byte-parity against legacy for the fields both sides have |

---

## 5. Suggested order and effort

1. **Step 1 (mapping)** — small, pure, unblocks everything; ~1 sitting. Do the G9
   decision here.
2. **Step 2 (wiring + test_id keys)** — medium; coordinate with the engine/merge work
   already tracked in cellpy-core.
3. **Step 4 legacy-import shim** — medium; needed early so real files can flow into
   the new model for testing.
4. **Step 3 (resolver + loader contract)** — medium-large (touches every loader, but
   each change is mechanical: "set attributes" → "fill draft").
5. **Step 4 new-format persistence** — depends on the cellpy-file container decision
   (sidecar JSON vs arrow metadata — decide together with the parquet format work).
6. **Steps 5–6** — small, after the above.

## 6. Open questions (decide before or during Step 1)

1. **G9:** tester software/calibration fields into `TestMeta` vs an `extra` dict.
   *Recommendation: into `TestMeta` — they are bounded, known, and test-dependent.*
2. **Units on meta quantities** (`mass`, `nom_cap`, …): legacy TODO suggests pint.
   *Recommendation: keep plain floats + a documented canonical unit per field in the
   spec (core stays pint-free per the units-by-value seam decision); revisit only if
   loaders genuinely deliver mixed units.*
3. **`CellMeta` placement:** separate `Data.meta` (this plan) vs only via
   `TestMeta.cell` links. The scaffolding doc defers this; this plan proposes the
   separate attribute because "one cell, many tests" is the dominant cellpy use case,
   with `TestMeta.cell` reserved for the exotic merged-different-cells scenario.
4. **Controlled vocabularies** for `test_family`/`test_type`/`source_type`: stay free
   text now; collect observed values during migration and revisit.
5. **Cellpy-file container format** (zip-of-parquet + `meta.json` vs single arrow
   file with schema metadata) — owned by the file-format work, but Step 4 needs the
   call.
6. **Identity and ontology parking (Stage-0, added 2026-07-09 from the gap
   analysis F10):** core's `column-headers-review.md` §F records the platform
   strategy needs — stable UUIDs for cells / electrodes / materials / protocols /
   test-runs, and BattINFO/EMMO term mappings per column. This plan covers
   `TestMeta.uuid` only; `CellMeta` has **no** uuid and no plan owns the wider
   identity scheme. Deliberately deferred, on this list so it is not forgotten:
   when the DB/API work (Step 7) gets a concrete consumer, add `CellMeta.uuid`
   plus the linkage strategy, and capture the ontology mapping alongside the
   column specs. Related #24 follow-up to fold in here: `cycle_type` likely
   migrates to per-test metadata as `test_type` — decide when populating
   `TestMeta.test_type` in Step 3.
7. **Per-test `raw_units`?** *(added 2026-07-09, cross-check vs
   unit-handling-cellpy2-plan.md)* This plan makes test-dependence first-class
   (`TestMetaCollection`, merged tests from different sources), but `raw_units` stays
   a single structure per `Data`. Tests merged from *different instruments* can have
   different raw units — today that case is silently unsupported. Decide: (a) keep one
   `raw_units` per `Data` and make `merge()` **reject** unit mismatches (cheap, honest,
   my recommendation for v2.0), or (b) move `raw_units` onto `TestMeta` per test and
   teach the engines per-test conversion factors (correct but engine-invasive; defer
   until a real mixed-unit merge use case exists). Note re open question 2: the unit
   plan resolves it the same way this plan recommends — plain floats in cellpy units,
   quantity-string coercion only at the API boundary via `cellpycore.units`.
