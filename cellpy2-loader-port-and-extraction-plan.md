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

### Decision (2026-07-10, issue #438) — curve-schema home

**`cellpycore.curves` + spec-first `CurveCols`** (`docs/data_format_specifications/curve_table.md`).
`get_cap` / `get_ccap` / `get_dcap` / `get_ocv` / `collect_capacity_curves` are pure frame
computations over raw+steps in core; cellpy 2 `CellpyCell.get_cap(...)` is a thin wrapper
(units/labels at the app layer). Gates [cellpy-core#118](https://github.com/cellpy/cellpy-core/issues/118).

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

### 2.4 Third-party extension API — loaders and exporters *(added 2026-07-09, gap G12)*

Contributions from users must not require a PR into cellpy itself. Two halves,
deliberately specified **before** Steps 3–5 hard-code the loader configurations:

**Loader entry points.** Plugin discovery is literally a TODO today
(`find_all_instruments`, data_structures.py:985: "Searching for modules through
plug-ins: Not implemented yet"). Define it now, cellpy-side:

- Entry-point group **`cellpy.loaders`**: a third-party package declares
  `[project.entry-points."cellpy.loaders"] myinstrument = "mypkg.loader"`; the loading
  framework discovers it lazily (first use, not import time — per the no-import-time-
  side-effects rule) and registers it through the existing `InstrumentFactory`.
- The §2.2 configuration schema is validated **at registration**, so a broken plugin
  fails loudly with the package named, and `validate_raw_frame` strict mode applies to
  plugin output exactly as to built-ins (no trust tier).
- `local_instrument` (YAML, no code) remains the entry-level path for simple text
  formats; the entry point is the path for real formats.
- **Contributor kit**: one docs page ("write a loader" = vendor parse + declaration +
  golden-fixture recipe from Step 0) and a cookiecutter template containing the
  test scaffolding. cellpy-core stays consumer-agnostic — discovery is app-layer only.

**Exporter contract.** There is no exporter framework, and none is needed — but the
*contract* must be written down: an exporter is a plain function

```python
def export(data: Data, target, *, cell_meta=None, units=None, **options) -> None
```

over the native `Data` (spec'd polars frames via `Schema`, dict-shaped meta via
`to_dict`, explicit unit specs) — no ABC tower, no registry requirement. Optionally,
an entry-point group **`cellpy.exporters`** lets the CLI/batch tools *discover*
third-party exporters, same lazy mechanics as loaders. The BDF exporter
(`cellpy-core/scripts/bdf/`, placement parked in `bdf-io-placement.md`) is promoted to
**reference implementation** once its home is decided; `CellpyCell.to_csv`/`to_excel`
eventually become thin wrappers over exporters that follow the same contract
(utils-plan territory, not this plan).

### 2.5 Harmonize policy decisions (2026-07-20, maintainer, cellpy#599)

Settled before the tier port so every ported loader starts from the same
rules:

1. **Unknown vendor columns are warn + drop.** `harmonize()` drops every
   undeclared column (that is what keeps the harmonized frame on-spec), and
   now warns once per load naming the unrecognised ones. Deliberate discards
   are listed in `LoaderDeclarations.dropped` and drop silently — a column
   cannot be both mapped and dropped (`LoaderError`). Rejected alternatives:
   keeping unknowns as prefixed aux columns (pollutes the specced frame),
   hard error (a firmware update adding one junk column would break
   ingestion).
2. **`maccor_txt_one`'s `Watt-hr` maps to charge energy.** The configuration
   double-claimed it for `power_txt` and `charge_energy_txt`; a Watt-hour
   column is dimensionally energy, so the power mapping was a slip. With
   cellpycore 0.2.3's native energy columns it derives to
   `cumulative_charge_energy`. Visible on the legacy path too (raw column
   renamed `power` → `charge_energy`); golden regenerated.

The port itself (tiers 1–2, `LegacyLoaderAdapter` removal) remains — this
section only fixes the ground it stands on.

### 2.6 Tier-3 execution record (2026-07-20, cellpy#561/#600)

- **`ext_nda_reader` — parked.** Its load path called a `load_nda()` stub that
  printed its arguments and returned `None`, so a "successful" load produced
  an empty cell; it also claimed `instrument_name = "neware_nda"`, colliding
  with the real fastnda-backed loader. Selecting it now raises a typed
  `LoaderError` naming the replacement (`instrument="neware_nda"`); removal
  in 2.1 unless users object.
- **`biologics_mpr`, `batmo_bdf` — port** (decision stands); executes with
  the tier-1/2 arc.

### 2.6b `arbin_res` two-stage (2026-07-21)

The tier-1 loader flagged as the awkward one (Access `.res` via ODBC on
Windows, mdbtools on posix). Two facts settled the scope:

- **CI can check it.** The arbin golden runs green on Linux CI — mdbtools is
  installed there — so the parity case runs in CI, not only locally. The
  parity test still skips gracefully where no backend exists, mirroring the
  golden's `skip_reason`.
- **The vendor stage is thin.** `get_headers_normal()` is already a
  `{cellpy attr → Arbin column}` map, the same shape the configuration loaders
  carry, so `declarations()` reuses `derive_column_maps` — Arbin gets the
  provenance rule (`Test_ID` dropped, not mapped onto the framework `test_id`)
  for free. `parse()` is just the database read, stopping before the rename and
  the Excel-serial datetime conversion, which `harmonize()` now owns. It shares
  the read path with `loader()` (calls `_loader_win`/`_loader_posix`), so the
  two cannot drift.

Scoped to a single test: the golden fixture is single-test with no aux columns,
and `loader()` still handles the multi-test split and aux joins. Multi-test
`.res` — where `Test_ID` is nonzero and `harmonize()`'s framework `test_id`
would differ from the vendor value — is the switchover's concern, not the
vendor stage's. Value parity holds on all 17 shared measurement columns
(capacities, energies, potential, current, cycle, step, times); only `date_time`
is excused, as everywhere.
- **`local_instrument` — confirmed** as one of the two sanctioned warn-only
  escape hatches (conventions plan §4); no change.

### 2.6c Tier-2 `arbin_sql` family — only `arbin_sql_h5` is verifiable (2026-07-21)

The tier-2 arbin_sql loaders (`arbin_sql`, `arbin_sql_7`, `arbin_sql_csv`,
`arbin_sql_xlsx`, `arbin_sql_h5`) share one shape: `BaseLoader` with a
module-level `normal_headers_renaming_dict`, so `declarations()` reuses
`derive_column_maps` exactly as `arbin_res` does. But **only `arbin_sql_h5` has
an in-repo fixture** — the other four read a live SQL Server or export formats
with no committed sample, so their declarations cannot be verified by the parity
oracle at all. Porting them blind would ship unverified column maps — the
failure mode this arc exists to prevent — so they are **left for the
switchover** (routed through `harmonize(declarations())` there, verified against
whatever real data exists then, or flagged unverified).

`arbin_sql_h5` is ported and CI-checked. Two facts about it:

- **`drop_duplicates` is a no-op on the fixture** (47 → 47), but it is genuine
  vendor cleanup (the export repeats rows), so `parse()` keeps it.
- **Internal resistance is measured sparsely** (35 of 47 rows null in the
  fixture) and forward-filled. Not declarative, so it is a declared post hook —
  the new reusable `hooks.forward_fill`. Removing the hook fails the parity case
  on `internal_resistance` (different null positions), so the check bites.

**Bug found, not fixed here.** The legacy `_post_process` runs a
`backward_fill_ir` step that calls `ffill` a second time instead of `bfill` (its
debug log even says "forward filling ir"), so the intended backward fill never
happens and leading nulls stay null. The port reproduces the *actual* behaviour
(forward-fill only) — its contract is no change to users' numbers — and the bug
is filed separately; fixing it would be a lock-step change to the port plus a
release note.

Also generalised here: the parity oracle loads its reference with
`auto_summary=False`, comparing the **loader** stage on both sides. Most loaders
are unaffected, but `arbin_sql_h5`'s `make_summary` prunes duplicate raw rows as
a documented side effect (47 → 34, issue #385, value-identical), and a
loader-stage frame must be compared against a loader-stage frame.

### 2.6a `pec_csv` two-stage decisions (2026-07-20, maintainer)

`pec_csv` is not configuration-driven, so its declarations cannot be *derived*
— they are hand-written. It differs from the config loaders in three ways that
forced explicit calls:

- It is a `BaseLoader` (no `config_params`, no renaming dict).
- Columns are identified by **alias set** (`_HEADER_ALIASES`), not a fixed
  vendor string: one semantic name matches many possible spellings.
- **Units are per column, embedded in the header** — `Voltage (V)` vs `(mV)`,
  four time formats — and scaled per column. `harmonize()` has no per-column
  unit scaling (it carries one `raw_units` for the whole frame), so leaving
  this to harmonize would tag mixed mV/V as V: silent corruption, the same bug
  class this arc keeps closing.

Decisions:

1. **Per-column unit scaling stays in `parse()`.** The vendor stage normalizes
   every column to the canonical unit, so the frame handed to `harmonize()` is
   uniform. Rejected: teaching `harmonize()` per-column source units — a large
   change to the shared path for one loader, unjustified until a second needs
   it.
2. **`parse()` renames matched columns to canonical semantic names**
   (`voltage`, `current`, …) and **`declarations()` is a static
   `{canonical → native}` map.** This puts `pec_csv` on the same footing as the
   config loaders — static declarations, `harmonize()` untouched — and collapses
   the three parallel alias/unit/header dicts toward one. Rejected: building
   declarations per file from the matched headers (workable, but keeps a
   per-file map for no gain now that units are already normalized in `parse()`).
3. **Keep `BaseLoader`; give `pec_csv` its own `parse()`/`declarations()`.**
   Reparenting to `AutoLoader` fights the alias design and buys nothing — pec's
   preamble sniffing and header matching are genuinely bespoke.
4. **Metadata parked.** `test_id`, `test_regime_name`, `lot_id` and the LotID
   multi-file grouping go to the metadata arc (#562/#563), like `date_time`.
5. **Cycle 0→1 shift kept** (it is inline today), routed through the shared
   `hooks.cycle_number_not_zero` for one implementation.

Verification is unchanged: `pec_csv` has no config-driven path, so the frame
`loader()` produces today *is* the oracle, and the existing `loader_pec_csv`
golden pins it.

**Execution (2026-07-21).** `parse()` and `declarations()` landed as decided;
`loader()` is untouched. Two things worth recording:

- The canonical→native map is **not hand-transcribed** — `declarations()`
  reuses `derive_column_maps` by inverting `_COLUMN_KEY_TO_CELLPY_HEADER`
  (canonical → legacy attr) and composing it with cellpy-core's legacy→native
  map. So `pec_csv` gets the provenance rule (`test` dropped, not mapped onto
  the framework `test_id`) and the 0.2.3 energy mapping for free, from the same
  code the config loaders use. Only the *passthrough* set is per file: whatever
  non-canonical columns the export carried, kept under their sanitised names,
  matching what `loader()` preserves.
- Building it surfaced a **false negative in the parity oracle**, not a pec
  bug: pec carries several sparse per-measurement columns that are all-null in
  this fixture (`cell_id`, `station_temperature_degc`, the DC/AC internal
  resistances). The numeric diff of two all-null columns is `None`, which the
  oracle had been counting as a mismatch. Worse, it would equally have hidden a
  real values-vs-null difference. The comparison now checks the null masks
  first, then the non-null values — so an all-null-vs-all-null column agrees,
  and a populated-vs-null column is caught. Mutation-checked both ways.

pec reaches value parity on every mapped and passthrough column; the only
excluded column is the shared `date_time` representation gap.

### 2.7 The post-processor map, and what the value oracle found (2026-07-20, cellpy#560)

The switchover replaces the legacy `post_processors` chain with `harmonize()`.
Every post-processor therefore needs a verdict. Establishing them is what the
value-parity oracle (`tests/test_loader_port_parity.py`) is for: it drives one
`cellpy.get()` per case with `query_file` wrapped, so the same run yields the
vendor frame *and* the legacy frame, derives declarations from the loader's
live `config_params`, and compares **values** rather than column names.

| Legacy post-processor | Verdict |
|---|---|
| `rename_headers` | `column_map` |
| `_convert_units` | `raw_units` |
| `select_columns_to_keep` | the harmonize select |
| `set_index` | n/a — polars frames carry no index |
| `convert_date_time_to_datetime` | `_convert_timestamps` (but see `date_time` below) |
| `cumulate_capacity_within_cycle` | **`ResetGranularity.PER_STEP`** — derived, see below |
| `convert_step_time_to_timedelta` / `convert_test_time_to_timedelta` | **`duration_columns`** — new framework feature |
| `split_capacity` / `split_current` | shared `hooks.state_splitter` — **ported** |
| `remove_last_if_bad` | `hooks.drop_last_row_if_worse` — **ported** |
| `update_headers_with_units` | resolved in the derivation, not a hook — **ported**, see below |
| `set_cycle_number_not_zero` | `hooks.cycle_number_not_zero` — **ported**, see the decision below |
| `remove_last_if_bad` | vendor post hook — **not yet ported** |

Two of these were found by the oracle rather than by reading, and both are the
kind that produce wrong numbers instead of errors. **Neither reached users:**
`declarations_from_configuration` has no production callers and `harmonize()`
is reached only by the `maccor_txt_native` pilot, so ingestion still runs the
legacy chain, which handles both cases correctly. The numbers below are diffs
against that legacy oracle — regressions the flag day would have introduced,
caught while the path was still behind a flag. Nothing needs re-loading.

1. **`cumulate_capacity_within_cycle` *is* `PER_STEP`.** It offsets each step by
   the running total of the cycle's completed steps — precisely what `PER_STEP`
   normalization undoes. Nothing said so, and the derivation defaulted every
   cumulative column to `PER_CYCLE`, so neware capacities came out wrong by up
   to 8 mAh against the legacy frame. The granularity is now **read from the
   configuration's `post_processors`** instead of defaulted: the configuration
   already knew, we were just not asking it.
2. **Duration strings silently became null columns.** Neware writes `Time` and
   `Cumulative Time` as `"00:01:00"`; the schema dtype for `step_time`/
   `test_time` is Float64; `_cast_to_schema` cast non-strictly, so all 9065 rows
   became null with no error. Fixed two ways — a declared `duration_columns`
   conversion (also derived, from the `convert_*_to_timedelta` flags), and a
   guard in `_cast_to_schema` that **raises when a cast empties a column
   entirely** while still tolerating stray junk rows with a warning. Partial
   tolerance is deliberate: the legacy path used
   `pd.to_numeric(errors="coerce")`, and making stray values fatal would refuse
   files 1.x loaded.

The second is the same failure shape as [#580](https://github.com/jepegit/cellpy/issues/580)
(Maccor zero capacities): a silent data-destroying step on the ingestion path.
The difference is only timing — #580 shipped, this was caught before the path
went live. It is worth stating as a rule for the rest of the port — **on the
ingestion path, a step that can destroy data must not be able to do it
quietly.**

**Status after this pass.** `neware_txt` reaches exact value parity on all 14
comparable columns. `maccor_txt` reaches parity on everything except the three
columns owned by the unported hooks above (`current`, the capacities, and
`cycle_num`). The oracle's exception list is derived from each configuration's
own `post_processors`, so deleting a row from the table above tightens the test
automatically — no column is ever excused silently.

**State splitting (2026-07-20, second pass).** `split_capacity` and
`split_current` are now the shared `hooks.state_splitter`, and `maccor_txt`
reaches value parity on `current` and both capacities. Three things the port
had to settle, all recorded because they are invisible from the code:

- **`split_capacity` changes the *shape* of the mapping**, not just values: one
  vendor column becomes two native ones. So the derivation drops the direct
  mapping, the hook synthesises two vendor-side columns, and those are mapped
  instead. It is the only post-processor that does this.
- **The split supersedes a competing direct mapping.** `maccor_txt_one` also
  declares a `Discharge_Capacity(Ah)` column (under a "not observed yet"
  comment). In 1.x `split_capacity` runs in the *unordered* pass — after
  `rename_headers` — so it overwrote whatever the rename produced. The
  derivation reproduces that precedence; without it the declarations fail
  validation for mapping two vendor columns onto one native column.
- **`propagate` is not a forward fill.** A direction's rows carry their own
  value, rows *after* that direction's last row in the cycle carry its final
  value, and everything else — including a rest *between* two same-direction
  rows — is zero. A forward fill would fill that middle rest. Physically the
  fill is arguably better; it is not what 1.x produced, so the port reproduces
  the quirk and pins it with a hand-computed test.

That last point caught a latent bug in the **#559 pilot**: its own
`split_capacity_by_state` was built on `forward_fill()` while its docstring
claimed to reproduce `_state_splitter`. The in-tree Maccor fixtures contain no
charge–rest–charge pattern within a cycle, so the divergence never showed. The
pilot now uses the shared splitter, so there is one state-splitting semantics
in the tree rather than two.

**Decision (2026-07-20, maintainer): 2.0 keeps 1-based cycle numbering.**
`set_cycle_number_not_zero` adds 1 to every cycle when the vendor's minimum is
0, and three Maccor dialects run it. Whether cycles start at 0 or 1 is a
user-visible contract — it reaches summary indices, plot axes and
`get_cap(cycle=N)` — so the port reproduces 1.x rather than adopting the
vendor's own numbering. Rejected alternative: accept vendor numbering as more
honest to the file, which is defensible but is a breaking change with no
forcing reason to take it now. Unlike the frame schemas this one **can** be
shimmed, so revisiting it in 2.1 stays open.

Its quirk is reproduced too: the shift applies only when the minimum is
*exactly* 0, so a file whose cycles start at 2 keeps starting at 2 rather than
being rebased to 1. The hook raises on a non-numeric cycle column instead of
silently doing nothing, since a string column would compare unequal to 0 and
leave every cycle index off by one with nothing to show for it.

**Parity status after this pass: the oracle's unported-post-processor exception
list is empty.** Both tier-1 loaders match the legacy path on every shared
column; the only remaining exclusion is the `date_time` representation gap
below.

**`update_headers_with_units` is not a post hook — and it exposed an
order-dependence (2026-07-20, third pass).** Neware spells its columns with the
unit baked in (`Current({{ current }})` → `Current(A)`), and the legacy
post-processor resolved those placeholders by **mutating `config_params` in
place at load time**. The mutation does not reach the configuration *module*,
because `ModelParameters` carries its own copy. So the derivation gave two
different answers for the same configuration:

| Derived from | Result |
|---|---|
| a post-load `config_params` | correct vendor names |
| the configuration module | **seven** names left as `Current({{ current }})`, matching no file, silently unmapped |

`test_derived_declarations` derives from modules, so it had been checking seven
fewer neware columns than it appeared to. Substitution now happens inside the
derivation, which makes it self-contained and order-independent, and a missing
unit label raises instead of interpolating the string `"None"` into an
unmatchable column name.

**Consequence: neware declarations cannot be static — resolved by making
`declarations()` a method (2026-07-20, fourth pass).** The units are read *from
the file* — the shipped configuration defaults to `mA`/`mAh`, while
`neware_uio.csv` is in `A`/`Ah`, and the loader updates `raw_units` at load time
to match. So the vendor column names, and therefore the `column_map` keys,
depend on the file being read; a `LoaderDeclarations` frozen at class level
cannot express that.

`AutoLoader` now exposes the two stages directly:

- **`parse(source, **kwargs)`** — the vendor stage, i.e. what `loader()`
  already did before it started building a `Data` (pre-processors, formatter
  parameters, `query_file`), exposed on its own.
- **`declarations()`** — derived from `config_params` *after* the parse, which
  is what makes it per-file with no loader having to opt in. It **raises if
  called before `parse()`**: reading it first would hand back vendor names no
  file contains, and those columns would be silently unmapped rather than
  erroring.

**This is not a change to the #210 Protocol, and 2.0 does not freeze it.**
`InstrumentLoader` requires `can_load` and `load` plus the three capability
attributes; `check_loader` checks those, the result types, the raw frame, units,
the meta draft and determinism. Neither `parse` nor `declarations` appears in
either. The two-stage split lives entirely behind `load()`, so it stays free to
change in 2.1 — worth stating plainly because the sequencing constraints
(§5.6 / the #560 issue) make "settle it before the Protocol freezes" the default
assumption for anything loader-shaped, and here that assumption does not apply.

**`remove_last_if_bad` — the denominator matters.** The legacy post-processor
ran after `rename_headers` **and** `select_columns_to_keep`, so it counted
missing values over the columns cellpy keeps. Hooks run before renaming, so the
derivation passes the declared vendor columns explicitly; counting over every
vendor column instead would let undeclared junk decide whether a real
measurement survives. It is the only shipped post-processor that changes the
row count. `maccor_002.txt` (added as a third parity case — it was in
`testdata` but no test loaded it) exercises the configuration but does **not**
trigger the drop, so the rule itself is pinned by hand-computed unit tests
while the parity case confirms it does not fire wrongly.

**A dtype normalization worth a release note.** Maccor model two leaves
`cumulative_charge_energy` as *strings* (`'0.00000000'`) on the legacy path;
harmonize casts it to Float64. Values verified identical (max abs diff 0.0,
every value coercible), so this is a representation improvement rather than a
change of numbers — users get a numeric column where 1.x gave text.

**Still open: `date_time`.** It survives as a passthrough string while the
native schema has no column for it, where the legacy path parsed it to datetime;
`epoch_time_utc` is not produced by either tier-1 configuration. This is a
metadata-arc question (#562/#563), not a capacity one, but it must be settled
before the flag day.

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
| 7 | **Extension API (§2.4)**: `cellpy.loaders` entry-point discovery + registration-time validation; exporter contract doc + optional `cellpy.exporters` discovery; contributor kit (docs page + cookiecutter); BDF promoted to reference exporter | 1 (config schema); BDF placement decision (`bdf-io-placement.md`) |

Each loader PR = configuration declarations + fixture + parity test; the framework
changes land once (Steps 1–2). Step 7's *contract* (entry-point group names, function
signature) is written in Step 1's PR even if the discovery mechanics land later — the
configurations must not bake in an assumption that all loaders live in-tree.

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
| Timezone semantics per vendor (Arbin local vs Neware device-time) | Per-loader `tz` declaration, defaulting to **naive = local + warn** (issue #438); recorded on `TestMeta.time_zone` |
| ODBC/Access dependency for `arbin_res` on CI | Keep the committed harmonized fixture as oracle (the core testdata approach — F8); loader test runs where the driver exists, fixture parity everywhere |
| Curves port drifts from legacy behavior in interpolation corner cases | Parity fixtures over multiple modes; document intentional differences (there are known quirks in forth-and-forth) |
| Three plans' loader changes colliding | This plan **is** the merge point; unit/metadata plans reference it instead of scheduling their own loader passes |

## 7. Effort

Framework + pilot ≈ 4–5 days; curves module ≈ 3–4 days (incl. spec); tier-1 ≈ 1 day
per loader; tier-2 ≈ 3–4 days total; tier-3 decisions ≈ 1–2 days. Total ≈ **3–4
focused weeks**, parallelizable per loader after Step 1.
