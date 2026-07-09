# Plan: redesign of the batch utility (batch v3)

**Date:** 2026-07-09
**Closes:** the design gap left open by the
[utils migration plan](cellpy2-utils-migration-plan.md) §2, which schedules
`utils/batch.py` + `batch_tools/*` for a wave-1 "port" but does not say what the
ported code should look like. This document is that design: a maintainable,
standard architecture for batch, plus the migration path from the current code.
**Foundation:** [usage report](data-and-cellpycell-usage-in-cellpy-utils.md),
[polars port plan](cellpy2-polars-port-execution-plan.md) (§1.3 keys-to-columns),
[metadata plan](cellpy2-metadata-handling-plan.md) (Step 4 journal IO, Step 6 db
feed), [conventions plan](cellpy2-conventions-plan.md) (deprecation policy).

---

## 1. Why redesign instead of port

The batch subsystem is ~11,000 lines across three layers that grew in
different eras and now overlap:

| Module | Lines | Role today |
|---|---|---|
| `utils/batch.py` | 2 232 | User-facing `Batch` class + 10 module-level entry points |
| `utils/batch_tools/batch_core.py` | 586 | `Doer`/`BaseExperiment`/`BaseJournal` abstractions ("farm/barn") |
| `utils/batch_tools/batch_experiments.py` | 1 148 | `CyclingExperiment` — the actual load/update workhorse |
| `utils/batch_tools/batch_journals.py` | 1 126 | `LabJournal` — data model + JSON/Excel IO + folder layout + selection UI |
| `utils/batch_tools/batch_plotters.py` | 1 544 | bokeh + matplotlib + plotly + seaborn summary plots |
| `utils/batch_tools/engines.py` | 412 | "engines" (half are `NotImplementedError` stubs) |
| `utils/batch_tools/` others | ~1 000 | exporters, analyzers, dumpers, helpers, reporters (23-line stub) |
| `utils/collectors.py` | 3 046 | Newer plotting/collection layer that overlaps batch_plotters |

A mechanical port would carry all of the structural problems below into
cellpy 2 and pay the migration cost twice. The subsystem is worth a redesign
because its *user-facing surface is small and beloved* (see §3) while its
*internals are large and unloved*.

### 1.1 The metaphor problem

The internal architecture is built on a farm metaphor: a `Doer` runs
`engines` that produce `farms` (lists of lists of DataFrames — the
"animals") which `dumpers` place in a `barn` (a string like `"batch_dir"`
that is resolved to an output directory at dump time). None of these names
communicate intent; new contributors must reverse-engineer the semantics
(`batch_core.py:19–141`, `engines.py:61`: *"Its a murder in the red barn"*).

### 1.2 Abstractions that never earned their keep

- `Doer` supports *lists* of experiments, engines and dumpers, but in
  practice `Batch` wires exactly one `CyclingExperiment` into one
  `CSVExporter` + one `BaseSummaryAnalyzer` + one `CyclingSummaryPlotter`
  (`batch.py:162–170`). The generality costs indirection every day and has
  never been used.
- `ImpedanceExperiment` and `LifeTimeExperiment` are empty stubs
  (`batch_experiments.py:1112–1118`); `cycles_engine`, `raw_data_engine`,
  `dq_dv_engine` warn or raise `NotImplementedError` (`engines.py:49–133`);
  `BaseReporter`/`batch_reporters.py` is a 23-line stub; `OriginLabExporter`
  is empty.
- The `Doer.__init__` appends **one** farm regardless of how many
  experiments are passed (`batch_core.py:57–60`), and the module-level
  `empty_farm = []` list is shared by reference — latent aliasing bugs that
  only stay dormant because nobody uses the multi-experiment path.

### 1.3 God-methods and duplication

`CyclingExperiment.update()` is ~380 lines
(`batch_experiments.py:206–585`) mixing argument merging from three sources
(journal row, `**kwargs`, `cell_specs`), file resolution, loader dispatch,
recalculation, per-cell export, progress-bar bookkeeping and error
collection. `parallel_update()` (`batch_experiments.py:586–908`) duplicates
most of it with a multiprocessing pool. Any bug fixed in one must be
remembered in the other.

### 1.4 API sprawl

Module-level entry points in `batch.py`: `init`, `init2`, `load`,
`load_journal`, `load_pages`, `naked`, `from_journal`, `from_journal2`,
`process_batch`, `iterate_batches` — plus `Batch(mode="old"/"new")` choosing
between `_init_old` and `_init_new` (which starts by logging *"Initializing
new-mode batch object"* — a mislabel, `batch.py:110`). Two half-finished
generations coexist with the original; users cannot tell which path is
blessed. There are 79 `kwargs.pop/get` calls in `batch.py` +
`batch_experiments.py` alone; keyword arguments tunnel four levels deep
(`load` → `Batch` → `update` → `cellpy.get` → loader) with precedence rules
documented only in one docstring.

### 1.5 Side effects and hidden coupling

- `csv_dumper` obtains output folders by calling `journal.paginate()` —
  which *creates directories on disk* as a side effect of exporting
  (`dumpers.py:22`).
- `update()` silently writes `nom_cap`/`instrument` into the journal pages
  (`batch_experiments.py:287–305`).
- The `Data` accessor mutates `experiment.cell_data_frames` during lookup
  and maintains a parallel `x_`-prefixed accessor dict for tab-completion;
  its inverse mapping uses `str.lstrip`, which strips *characters* rather
  than a prefix — a cell named `xenon_cell` round-trips to `enon_cell`
  (`batch_core.py:179–180`). This is a real bug today.
- Engines cache into `experiment.summary_frames`; plotters read it; reset
  semantics are implicit.

### 1.6 Everything in one place

`Batch` itself carries journal management, folder layout, loading, QC
checks (`_check_cell_*`, `batch.py:340–430`), styled reporting, CSV/Excel
export, journal duplication, cellpy-file copying and four plotting
backends. `LabJournal` similarly mixes the data model, three file formats,
path fixing, cell selection state and folder generation. Both are
change-magnets: every feature request lands in the same two files.

---

## 2. Design principles

1. **Boring and standard.** Data model / IO / orchestration / outputs are
   separate modules with names that say what they do. No metaphors.
2. **Functions over frameworks.** An "engine" becomes a function
   `cells -> frame`; a "dumper" becomes a function `frame, path -> None`.
   Composition replaces the `Doer` class hierarchy. Pluggability, where
   actually needed, is a list of callables — not an abstract base class.
3. **Per-cell work is a pure function.** One cell in, one result out, no
   shared mutable state. Serial vs parallel execution is then a choice of
   executor, not a second 300-line method.
4. **Errors are data.** A batch run returns a result object with per-cell
   outcomes; printing/raising is the caller's policy (`strict=True`).
5. **Typed options instead of kwargs tunnels.** One `LoadPolicy` dataclass
   replaces `force_cellpy`/`force_raw`/`force_recalc`/`accept_errors`/... ;
   one resolved `CellSpec` per cell replaces the three-way dict merge.
6. **The journal is a document, not an actor.** Reading a journal never
   touches the filesystem layout; creating folders is an explicit call.
7. **Batch produces tidy frames; plotting lives elsewhere** (collectors /
   plotutils, per the utils migration plan wave 2). Batch keeps only a thin
   `plot()` convenience that delegates.
8. **Keep the notebook UX.** `b.pages`, `b.cells["name"]`, `b.summaries`,
   lazy loading, tab completion — the parts users love survive unchanged or
   improve.

---

## 3. The user contract (what must keep working)

From docs, tutorials and the usage report, the essential workflow is:

```python
b = batch.load(name, project)        # or from journal file / db / frame
b.pages                              # journal as a dataframe
b.update()                           # load all cells (raw or cellpy files)
b.data["20231115_..."]               # lazy access to one CellpyCell
b.summaries                          # combined summary frame
b.plot()                             # summary overview plot
b.save()                             # persist journal + combined frames
```

Everything else (`mark_as_bad`, `drop`, `report`, `combine_summaries`,
`export_journal`, `link`, `recalc`, `duplicate_cellpy_files`, ...) is
secondary surface used interactively. The redesign keeps this contract on a
facade while replacing what is underneath.

---

## 4. Proposed architecture

New package `cellpy/batch/` (name free of the `utils.batch_tools` baggage;
`cellpy.utils.batch` becomes a shim, §6):

```
cellpy/batch/
    __init__.py      # public API: load(), from_journal(), Batch
    journal.py       # Journal model + json/excel readers/writers
    db.py            # journal-from-database (wraps existing db readers)
    policy.py        # LoadPolicy, CellSpec (typed options)
    runner.py        # load_cell(spec) -> CellResult; run(journal, policy) -> BatchResult
    result.py        # CellResult, BatchResult (outcomes, errors, timings)
    store.py         # CellStore: lazy Mapping[str, CellpyCell]
    aggregate.py     # combine summaries/steps into tidy frames (polars)
    qc.py            # per-cell checks -> one tidy QC frame (from _check_cell_*)
    layout.py        # BatchPaths: pure path computation + explicit ensure_dirs()
    outputs.py       # write_csv/write_excel/write_parquet (pure writers)
    facade.py        # Batch class: thin, notebook-friendly wrapper over the above
```

Target size: **≤ 2 500 lines total** (vs ~8 100 today, before counting
collectors) — the farm machinery, the duplicated update paths, the stubs and
two of the four plotting backends account for the difference.

### 4.1 Journal (`journal.py`)

A plain data model, serialization separated from the model:

```python
@dataclass
class Journal:
    name: str
    project: str
    pages: pl.DataFrame          # one row per cell; HeadersJournal names
    session: Session             # bad_cells, notes, versions ...
    meta: JournalMeta            # time_stamp, format version ...

def read_journal(path) -> Journal            # .json (+ legacy formats via shim)
def write_journal(journal, path) -> None
def journal_from_db(name, project, reader=...) -> Journal
def journal_from_frame(frame, ...) -> Journal
```

- No `paginate()` on the journal. Folder layout moves to `layout.BatchPaths`
  (pure computation) + `ensure_dirs(paths)` (the only function that mkdirs).
- Selection state (`select_group`, `mark_as_bad`, ...) becomes plain column
  operations on `pages` exposed as small helpers on the facade; the journal
  file format keeps a `selected`/`bad` column so nothing is lost.
- Pages move keys-to-columns per the polars report §1.3 (cell label is a
  column, not an index).
- JSON format: keep the current `cellpy_batch_*.json` schema readable
  (versioned, with a legacy loader for the old shape); Excel journal support
  is read-only from day one and deprecated per the metadata plan Step 4.

### 4.2 Typed options (`policy.py`)

```python
class SourcePreference(StrEnum):
    AUTO = "auto"                # cellpy file if fresh, else raw (today's default)
    CELLPY_ONLY = "cellpy_only"  # replaces force_cellpy=True
    RAW_ONLY = "raw_only"        # replaces force_raw_file=True

@dataclass
class LoadPolicy:
    source: SourcePreference = SourcePreference.AUTO
    recalc: bool = False                  # replaces force_recalc
    max_cycle: int | None = None
    accept_errors: bool = True            # errors collected, not raised
    all_in_memory: bool = False
    skip_bad_cells: bool = False
    selector: dict | None = None          # forwarded to cellpy file loader
    loader_kwargs: dict = field(default_factory=dict)  # the *one* escape hatch

@dataclass
class CellSpec:
    """Fully resolved per-cell loading instructions."""
    label: str
    raw_files: list[OtherPath]
    cellpy_file: OtherPath | None
    instrument: str | None
    model: str | None
    mass: float | None
    nom_cap: float | None
    area: float | None
    cycle_mode: str | None
    overrides: dict            # from cell_specs / kwargs, applied last
```

`resolve_specs(journal, policy, per_cell_overrides) -> list[CellSpec]` is
the *single* place where journal-row values, policy and per-cell overrides
merge (precedence: journal < policy-level overrides < per-cell). Today that
logic is smeared across 200 lines of `update()`; here it is a pure,
unit-testable function.

### 4.3 Runner (`runner.py`)

```python
def load_cell(spec: CellSpec, policy: LoadPolicy) -> CellResult:
    """Pure-ish: loads one cell (raw or cellpy file), optionally recalcs,
    returns the cell or the exception. No prints, no journal mutation."""

def run(journal, policy, executor="serial", on_progress=None) -> BatchResult:
    specs = resolve_specs(journal, policy)
    results = EXECUTORS[executor](load_cell, specs, policy, on_progress)
    return BatchResult(results)
```

- `executor="serial" | "processes" | "threads"` — the parallel path reuses
  the same `load_cell`, deleting the 320-line `parallel_update` clone.
- Progress reporting is a callback (the facade passes tqdm); the engine
  layer never imports tqdm or prints.
- `BatchResult` carries per-cell status, wall time, source used
  (raw/cellpy), and the exception if any. `result.raise_if_failed()` gives
  strict mode; `result.report()` gives the dataframe the current `errors`
  list only hints at.

### 4.4 Cell store (`store.py`)

```python
class CellStore(Mapping[str, CellpyCell]):
    """Lazy dict of cells; loads from cellpy files on first access."""
    def first(self): ...
    def sample(self): ...
    def unload(self, label): ...        # explicit memory management
```

Replaces `batch_core.Data` + `cell_data_frames` + the `x_` accessor dict.
Tab completion comes from `_ipython_key_completions_` (the standard
mechanism for `store["<TAB>"]`) instead of prefix-mangled attribute names —
which also removes the `lstrip` bug (§1.5). `b.data` on the facade stays as
a deprecated alias for `b.cells`.

### 4.5 Aggregation, QC, outputs

- `aggregate.combine_summaries(cells, journal) -> pl.DataFrame`: one tidy
  long-format frame with `cell`, `group`, `sub_group` columns (replaces the
  `join_summaries` wide/multiindex machinery — plotting code downstream
  gets dramatically simpler, cf. collectors).
- `qc.check(cells, journal) -> pl.DataFrame`: the `_check_cell_*` family as
  one function returning a tidy pass/fail frame; the styled `report()` on
  the facade renders it.
- `outputs.write_*`: pure writers taking a frame and an explicit path from
  `BatchPaths`. Exporting never creates directory trees implicitly.

### 4.6 Facade (`facade.py`)

```python
class Batch:
    journal: Journal
    cells: CellStore
    policy: LoadPolicy

    def update(self, **overrides) -> BatchResult: ...   # delegates to runner.run
    @property
    def pages(self) -> pl.DataFrame: ...
    @property
    def summaries(self) -> pl.DataFrame: ...            # cached aggregate
    def plot(self, backend="plotly", **kwargs): ...     # delegates to collectors
    def report(self, check=False): ...
    def save(self): ...                                 # journal + combined frames
    def mark_as_bad(self, label): ...
    def drop(self, label): ...
```

One public constructor path:

```python
def load(name=None, project=None, *, journal=None, db=None, frame=None,
         policy=None, **legacy_kwargs) -> Batch
```

`init`, `init2`, `naked`, `from_journal2`, `process_batch`,
`iterate_batches` become deprecated thin wrappers (§6); `from_journal`
stays as the second blessed entry (it is widely used and reads well).

### 4.7 Plotting

`batch_plotters.py` is not ported. The facade's `plot()` calls the
collectors/plotutils layer (wave 2 of the utils plan) with the tidy
`summaries` frame. Backend triage: **plotly** (primary), **matplotlib**
(kept for print/CLI), **seaborn** (drop — matplotlib covers it), **bokeh**
(drop — unmaintained path, big dependency). This needs a maintainer
sign-off since bokeh output is visible in old tutorials.

---

## 5. What this fixes, concretely

| Today | After |
|---|---|
| farm/barn/engine/dumper vocabulary | journal/policy/runner/result/store — self-describing |
| `update()` 380 lines + `parallel_update()` clone | `load_cell()` + executor choice; one code path |
| 10 entry points, 2 generations | `load()` + `from_journal()`; rest deprecated shims |
| kwargs tunnels, 79 pops | `LoadPolicy` + `CellSpec` with documented precedence |
| export creates folders via `journal.paginate()` side effect | `layout.ensure_dirs()` is the only mkdir |
| errors printed, partially collected | `BatchResult` with per-cell outcome + `raise_if_failed()` |
| `x_` accessor + `lstrip` label mangling bug | standard `Mapping` + IPython key completion |
| 4 plotting backends inside batch | tidy frames out; plotting delegated to collectors |
| stubs (`ImpedanceExperiment`, `raw_data_engine`, ...) | deleted, not ported |
| pandas MultiIndex summary joins | polars long-format with key columns |

---

## 6. Migration plan

Aligned with the utils migration plan wave 1 (batch is the wave-1 anchor)
and the conventions plan's deprecation policy. The redesign lands on the
cellpy2 contract directly (polars pages/frames, native summary names through
the aggregation layer) — one migration per module, no "structure now,
polars later" double pass.

### Phase 0 — characterization net (before any new code)

1. Extend the **batch end-to-end golden test** from the utils plan §5:
   journal → load → update → summaries → QC report for a two-cell batch
   (test data exists in `cellpy-core-dev/data`), asserting frame values,
   not just shapes.
2. Journal round-trip tests: current JSON (both shapes) and Excel → model →
   JSON, byte-stable where possible.
3. Inventory actual usage: grep docs, tutorials, example notebooks and any
   reachable user code for `batch.` attribute access; the hit list defines
   the facade's must-keep surface (seed list in §3).

*Effort: ~1 day. Exit: green golden tests against the old implementation.*

### Phase 1 — build `cellpy/batch/` alongside the old code

Order of construction (each step lands with its unit tests):

1. `journal.py` + `layout.py` (the contract; legacy JSON loader included).
2. `policy.py` + `resolve_specs` (property-tested precedence).
3. `runner.py` + `result.py` serial executor; `store.py`.
4. `aggregate.py`, `qc.py`, `outputs.py`.
5. `facade.py` + `__init__.py`.
6. Golden tests from Phase 0 run against **both** implementations; shared
   columns must agree (mapped through the header mapping where names
   changed).
7. Process-pool executor last (it is an optimization, not a blocker).

The old `utils/batch.py` is untouched throughout; both coexist.

*Effort: ~5–7 days. Exit: new API passes the golden parity tests.*

### Phase 2 — switch the plumbing, keep the sockets

1. `cellpy.utils.batch.Batch` and `load`/`init`/`from_journal` re-implemented
   as shims over `cellpy.batch` (constructor signatures preserved; exotic
   kwargs mapped to `LoadPolicy` fields or raising a helpful
   `DeprecationWarning` naming the replacement).
2. Symbol map published in the docs (old → new), e.g.:
   `b.experiment.journal.pages` → `b.pages`;
   `force_cellpy=True` → `policy.source=CELLPY_ONLY`;
   `b.update(cell_specs=...)` → `b.update(overrides=...)`;
   `combine_summaries()` → `b.summaries` (cached).
3. `batch_tools` modules gain module-level `DeprecationWarning` on import.
4. Docs/tutorials/notebooks in the cellpy repo updated to the new API (this
   is also the real-world test of the facade).

*Effort: ~3–4 days. Exit: cellpy test suite green with shims active;
tutorials run on the new path.*

### Phase 3 — dependent modules

1. `collectors.py` consumes `BatchResult`/tidy summaries instead of poking
   `experiment.summary_frames` (wave 2 work, but the interface lands here).
2. `helpers.concatenate_summaries` → thin wrapper over `aggregate` or
   deprecated.
3. Delete dead code immediately (stub experiments, stub engines,
   `batch_reporters.py`, `OriginLabExporter`, `do2`, `from_file_old`,
   `_old_duplicate_journal`, `_init_old`/`_init_new` split): none of it is
   reachable through the shims.

*Effort: ~2 days (excluding the wave-2 collectors port itself).*

### Phase 4 — removal

Per the conventions plan: shims warn in 2.0, `utils/batch_tools/` is
removed in 2.1. The `cellpy.utils.batch` import path itself can stay
permanently as a one-line re-export of `cellpy.batch` (cheap, avoids
breaking a decade of notebooks).

### Suggested sequencing relative to other plans

```
native-headers Phase 3/4 ──▶ batch Phase 0–1 ──▶ Phase 2 ──▶ Phase 3
                                   │                             │
                 (defines the utils contract, wave 1)     collectors (wave 2)
```

Total: **~2.5 weeks** focused work, matching the wave-1 + part of wave-2
budget in the utils plan (batch was 4–6 days of a mechanical port; the
redesign costs ~1 week more and removes ~6 000 lines).

---

## 7. Risks

| Risk | Mitigation |
|---|---|
| Hidden user reliance on `batch_tools` internals (custom engines/dumpers in the wild) | Phase 0 inventory; shims keep `Doer` importable-with-warning for one minor release; announce in release notes with the symbol map |
| Journal JSON incompatibilities | Versioned schema; legacy loader tested against real journal files collected from the maintainers' projects |
| Behavioral drift in load precedence (journal vs kwargs vs cell_specs) | `resolve_specs` unit tests encode today's documented precedence *before* refactoring; golden parity test |
| Dropping bokeh/seaborn upsets someone | Maintainer decision up front (§4.7); plot() backend error message points to collectors |
| Excel journals still needed by lab workflows | Read-only support kept; write path already targeted by metadata plan Step 4 |
| Parallel executor on Windows (pickling CellpyCell) | Results return frames + paths rather than live objects when `executor="processes"`; same design as today's `parallel_update` but explicit |

---

## 8. Open questions for the maintainer

1. **Bokeh and seaborn in batch plots** — drop as proposed? (plotly +
   matplotlib remain.)
2. **`iterate_batches` / `process_batch`** — CLI-ish helpers: keep as thin
   recipes in docs, or as maintained API?
3. Should the new package be `cellpy.batch` (proposed) or live inside
   `cellpy.utils`? `cellpy.batch` reads better and matches its status as a
   primary user surface.
4. Is multi-experiment batching (the thing `Doer` theoretically supported)
   a real future need? If yes, it becomes "a list of `Batch` objects" +
   cross-batch helpers in collectors — still no framework required.
5. Journal `session` contents (bad cells, notes) — fold into the metadata
   plan's resolver model now or keep as-is until Step 6 lands?
