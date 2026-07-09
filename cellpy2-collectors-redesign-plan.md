# Plan: redesign of the collectors util (collection as a first-class product)

**Date:** 2026-07-09
**Closes:** the third leg of the utils redesign triad. The
[batch redesign plan](cellpy2-batch-redesign-plan.md) owns loading and basic
aggregation; the [plotting redesign plan](cellpy2-plotting-redesign-plan.md)
owns every figure and already re-bases collectors' *drawing* half (§3.3).
This document owns what remains — the **collection layer**: gathering
per-cell data products across a batch, transforming them, and persisting
them with provenance.
**Foundation:** [utils migration plan](cellpy2-utils-migration-plan.md)
(collectors = wave 2, on `cellpycore.curves`),
[loader plan](cellpy2-loader-port-and-extraction-plan.md) §2.3,
[conventions plan](cellpy2-conventions-plan.md).

---

## 1. What collectors actually is today

`utils/collectors.py` is 3 046 lines, but the *collection* logic it owns is
small; the file is mostly plotting (~1 300 lines, claimed by the plotting
plan) plus orchestration machinery. The real stack:

```
BatchSummaryCollector / BatchICACollector / BatchCyclesCollector
        │  (argument "elevation", naming, autorun)
        ▼
BatchCollector (collectors.py:236–865)
        │  collect() ──► data_collector callable
        │  render()  ──► plotter callable          ◄── dies (plotting plan §3.3)
        │  save()    ──► csv + hdf5 + png/svg/json
        ▼
summary_collector ──► helpers.concat_summaries     (helpers.py:950, ~30 params,
cycles_collector  ──► c.get_cap per cell            the actual heavy lifting)
ica_collector     ──► ica.dqdv per cell
```

Line inventory of the affected code:

| Piece | Lines | Fate |
|---|---|---|
| `BatchCollector` + 3 subclasses + `standard_gravimetric_collector` | ~1 250 | redesigned here |
| collector functions (`pick_named_cell`, `summary/cycles/ica_collector`) | ~230 | redesigned here |
| plotter functions + fig IO + templates in collectors.py | ~1 300 | plotting plan |
| `helpers.concat_summaries` + private helpers | ~650 (of helpers.py's 1 628) | moves here |
| `helpers.concatenate_summaries` (deprecated twin, still shipped) | ~260 | deleted |

### 1.1 The good parts (this module invented the tidy contract)

- The collected frames are **long-format with `cell`, `group`, `sub_group`
  columns** — exactly the tidy contract that the batch plan (§4.5) and the
  plotting plan (§2.3) now standardize on. Collectors got there first.
- Rate-based cycle selection (`rate`, `rate_on`, `rate_std`, `inverse`),
  group averaging with std, equivalent-cycle normalization, CV
  partitioning — real, battery-specific features with no generic
  replacement.
- The save bundle (data + figure + plotly JSON side by side, serial-numbered
  filenames) matches how the maintainers actually produce reports.
- The hook system (`individual_summary_hooks`,
  `concatenated_summary_hooks`) proves there is genuine demand for user
  extension points.
- `pick_named_cell`'s label-mapper / journal-label renaming is a good idea
  (presentation names ≠ file names).

### 1.2 The problems

**The "elevated arguments" machinery.** Every subclass re-declares ~20 of
the underlying pipeline's parameters as its own keyword arguments
(`BatchSummaryCollector.__init__`, `collectors.py:995–1126`), packs them
into dicts, and merges them through **three priority layers** (subclass
defaults → `data_collector_arguments`/`plotter_arguments` dicts → elevated
kwargs; `collectors.py:306–316, 481–529`). The same parameter is therefore
declared and documented in three places (`concat_summaries` docstring, the
collector docstring, the defaults dict) with nothing keeping them in sync.
`None` is overloaded as "not set" so a user cannot elevate an explicit
`None`.

**Two summary pipelines ship simultaneously.**
`helpers.concatenate_summaries` (`helpers.py:683`, warns "not maintained
anymore") and `helpers.concat_summaries` (`helpers.py:950`) are ~90 %
identical ~17-parameter siblings; the live one has grown hooks,
recalc-kwargs, extreme-value clipping and CV partitioning. The deprecated
one is still importable and still ~260 lines.

**A real cross-cell bug in the curve collectors.** In `cycles_collector`
(`collectors.py:1607–1615`) and `ica_collector` (`collectors.py:1688–1696`)
the `cycles` list is *reassigned inside the per-cell loop*
(`cycles = list(set(filtered_cycles) & set(cycles))`). Each cell's
rate-filter result narrows the selection for **every subsequent cell** — if
cell 1 lacks cycle 7 at the requested rate, cycle 7 is silently dropped for
cells 2..N too. Order-dependent, silent, wrong.

**Side-effectful construction.** `BatchCollector.__init__` runs the full
collect + render pipeline by default (`autorun=True`,
`collectors.py:329–330`) and calls `parse_units()`
(`collectors.py:554–578`) which **iterates the whole batch** — lazily
loading every cellpy file from disk — just to warn about non-homogeneous
units. Instantiating a collector on a large batch can mean minutes of I/O
before the constructor returns.

**Error handling by print.** `update()` catches `TypeError` and prints
"Hint: fix it and then re-run using reset=True"
(`collectors.py:615–632`); `_check_plotter_arguments` translates
`plot_type` → `method` with a printed WARNING (`collectors.py:531–537`);
`_output_path` silently falls back to the current working directory when
the figure directory does not exist (`collectors.py:854–859`); the
wide-format CSV pivots are wrapped in bare try/except-print
(`collectors.py:1179–1205`).

**The "temporary hack" that became API.** `standard_gravimetric_collector`
(`collectors.py:867–978`) builds its report by constructing renamed
`functools.partial` objects and threading them through the hook system —
110 lines to express "the standard plot no. 1", self-described as a hack.

**Duplication with plotutils** (fig IO, templates, legend helpers,
`_image_exporter_plotly` subprocess dance) — inventoried in the plotting
plan §1.1 and removed there.

---

## 2. Design principles

1. **A collection is a product, not a side effect.** The output is a
   `Collection`: a tidy frame plus provenance (what was collected, from
   which batch, with which options, by which cellpy version, when). It can
   be saved, re-loaded and re-plotted without re-collecting.
2. **One source of truth per option set.** Options are dataclasses shared
   by the pipeline function and the convenience class. No elevation, no
   triple documentation, no three-layer merge.
3. **Collect and draw are different verbs.** Collectors produce frames;
   `cellpy.plotting` draws them (plotting plan §3.3). The convenience class
   wires them together but contains neither.
4. **Constructors construct.** Nothing loads gigabytes or renders figures
   inside `__init__`. Explicit `.collect()`, `.plot()`, `.save()` — with a
   one-line `collect_and_plot()` convenience for the notebook flow.
5. **Per-cell iteration is one shared generator** with per-cell isolation:
   one cell's filter results never leak into the next (fixes §1.2 bug 3).
6. **Hooks become transforms.** `transforms: list[Callable[[frame], frame]]`
   applied in order after collection — the formalized successor of the two
   hook lists, and the mechanism recipes like the standard report use.
7. **Errors raise; messages guide.** Bad options raise with the valid
   choices listed. Fallbacks (missing directory → cwd) become errors.

---

## 3. Proposed architecture

New package `cellpy/collect/` (import shim kept at
`cellpy.utils.collectors`, naming decision in §6):

```
cellpy/collect/
    __init__.py      # collect_summaries(), collect_cycles(), collect_ica(),
                     # load_collection(), BatchCollector (+ subclass aliases)
    options.py       # SummaryOptions, CurveOptions, IcaOptions, SaveOptions
    cells.py         # iter_cells(b, only_selected, label_mapper) -> CellItem
                     #   (successor of pick_named_cell; per-cell isolation)
    summary.py       # successor of helpers.concat_summaries: rate filtering,
                     #   normalization, grouping/averaging, CV partition —
                     #   built on batch.aggregate's combined frame
    curves.py        # voltage-capacity curves via cellpycore.curves
                     #   (fallback: c.get_cap until curves lands)
    ica.py           # dq/dv collection via utils.ica / cellpycore
    collection.py    # Collection dataclass; save/load (parquet + csv + meta.json,
                     #   hdf5 kept as legacy writer); wide-layout export
    transforms.py    # normalize/scale/clip/relabel building blocks + recipes
                     #   (standard_gravimetric becomes ~15 declarative lines)
    collector.py     # BatchCollector: thin class = options + collect + plot + save
```

Target: **~900–1 000 lines** for the package (from ~2 100 lines of
collection-side code today, after the plotting half already moved out).

### 3.1 The core object

```python
@dataclass
class Collection:
    data: pl.DataFrame            # tidy: cell, group, sub_group, cycle, ...
    kind: str                     # "summary" | "cycles" | "ica" | custom
    name: str                     # autogenerated (nick + kind + option tags)
    meta: CollectionMeta          # batch name, options used, cellpy version,
                                  # created-utc, cells included/skipped

    def plot(self, **plot_options): ...   # -> cellpy.plotting (frame carries
                                          #    cell/group -> multi-cell figure)
    def save(self, directory=None, formats=("parquet", "csv")) -> list[Path]
    def to_wide(self) -> pl.DataFrame     # explicit, tested pivot

def load_collection(path) -> Collection   # frame + meta.json round-trip
```

Provenance in `meta.json` makes collections reproducible artifacts: six
months later you can see exactly which rate filter and normalization
produced the figure in the report — and `Collection.plot()` re-renders it
without touching the raw data.

### 3.2 Collector functions (the pipeline)

```python
def collect_summaries(b, options: SummaryOptions | None = None,
                      **overrides) -> Collection:
    opts = (options or SummaryOptions()).replace(**overrides)
    frame = batch.aggregate.combine_summaries(b)        # batch plan §4.5
    frame = summary.filter_rates(frame, opts)           # per-cell isolated
    frame = summary.normalize(frame, opts)
    frame = summary.group_average(frame, opts)          # mean/std long-format
    for t in opts.transforms:
        frame = t(frame)
    return Collection(frame, kind="summary", meta=..., name=...)
```

`collect_cycles` / `collect_ica` follow the same shape over
`iter_cells` + `cellpycore.curves` / `ica`. Rate-based cycle selection is
computed **per cell from the originally requested cycles** (bug fix), and
per-cell failures are recorded in `meta.cells_skipped` instead of
`print`/abort (with `strict=True` to raise — mirroring the batch plan's
error philosophy).

### 3.3 The convenience class (what notebooks see)

```python
class BatchCollector:
    def __init__(self, b, kind="summary", options=None, autorun=True, **overrides):
        ...                       # stores options; autorun still collects+plots
                                  # (kept for UX parity) but does NOT touch
                                  # the filesystem and does NOT eagerly load
                                  # cells for unit checks
    def collect(self) -> Collection
    def plot(self, **plot_options)         # delegates to plotting package
    def save(self, **save_options)         # data + figure bundle, serial numbers
```

`BatchSummaryCollector(b, ...)`, `BatchICACollector(b, ...)`,
`BatchCyclesCollector(b, ...)` remain as one-line subclasses/aliases pinning
`kind` — existing notebooks keep working. Unit-homogeneity checking happens
during `collect()` (when cells are loaded anyway) and lands as a warning in
`meta`, not as constructor I/O.

### 3.4 Recipes replace the hack

```python
# transforms.py
def standard_gravimetric(norm_factor=120.0) -> SummaryOptions:
    return SummaryOptions(
        columns=[...],
        partition_by_cv=True,
        group_it=True,
        transforms=[normalize_column("discharge_capacity_gravimetric", norm_factor),
                    normalize_column("charge_capacity_gravimetric_cv", norm_factor)],
        plot_defaults=dict(spread=True, order_variables=[...]),
    )

b_std = BatchCollector(b, options=standard_gravimetric(norm_factor=140))
```

Declarative, testable, and the pattern users can copy for their own
standard reports (the "added functionality" analogue of the plotting plan's
`register_family`).

### 3.5 What dies

| Today | Fate |
|---|---|
| elevated-arguments machinery (3 priority layers, per-subclass re-declaration) | options dataclasses; `.replace(**overrides)` |
| `helpers.concatenate_summaries` (deprecated twin) | deleted (2.0) |
| `helpers.concat_summaries` | logic moves to `collect/summary.py`; helper kept as thin deprecated wrapper for one release |
| cross-cell `cycles` narrowing in curve/ica collectors | fixed by design (per-cell selection) |
| `parse_units` full-batch load in `__init__` | unit check during collect, result in meta |
| `plot_type`→`method` print-translation, TypeError-print-hints | raise with valid options listed |
| `_output_path` cwd fallback | explicit error; `SaveOptions.directory` |
| wide-CSV try/except pivots | `Collection.to_wide()`, tested |
| plotter functions, fig IO, templates in collectors.py | plotting plan (already scheduled) |
| hdf5 as default save format | parquet default; hdf5 writer kept one release for compat |

---

## 4. Migration plan

Sequenced **after** batch redesign Phases 0–2 (needs `batch.aggregate` and
`CellStore` iteration) and interlocking with plotting plan Phase 4 (which
deletes collectors' plotters). The collection side can start as soon as
`batch.aggregate` exists; it does not wait for the plotting backends.

### Phase 0 — characterization (½ day)

1. Golden collect tests: `summary_collector`, `cycles_collector`,
   `ica_collector` frames for a two-cell batch from `cellpy-core-dev/data`
   (values, not shapes), including a `group_it=True` case and a rate-filter
   case. **Encode the cross-cell narrowing bug's *corrected* expectation as
   an xfail-then-fix pair** so the fix is visible and deliberate.
2. Inventory which elevated arguments are actually used in docs, tutorials
   and maintainers' notebooks → the options-dataclass field list.

### Phase 1 — options + pipeline functions (2–3 days)

`options.py`, `cells.py`, `summary.py` (ported from `concat_summaries`,
minus the deprecated twin), `curves.py`, `ica.py`, `collection.py`.
Golden parity gate against the old collector functions (mapped through
header renames where applicable). Bug fix lands here with its test.

### Phase 2 — the class, re-based (1–2 days)

New `collector.py`; old `BatchCollector` and subclasses become shims
mapping elevated kwargs → options fields (unknown kwargs raise with the
field list). `standard_gravimetric_collector` re-implemented as the §3.4
recipe; old callable delegates. Constructor I/O removed.

### Phase 3 — plotting handover (with plotting plan Phase 4)

`Collection.plot()` and `BatchCollector.plot()` wired to
`cellpy.plotting`; collectors' plotter functions, fig IO and templates
deleted per the plotting plan. The save bundle (figure files) goes through
`plotting.figures`.

### Phase 4 — removals (2.x cadence)

`concatenate_summaries` gone in 2.0; `concat_summaries` wrapper and the
elevated-kwarg shims gone in 2.1; `cellpy.utils.collectors` import path
stays as a permanent re-export of `cellpy.collect`.

Total: **~1–1.5 weeks**, of which the genuinely new code is small — most of
the effort is porting `concat_summaries`' feature set faithfully onto the
options model.

---

## 5. Risks

| Risk | Mitigation |
|---|---|
| `concat_summaries` has many rarely-used options (hooks, recalc-kwargs, extreme clipping) that tests do not cover | Phase 0 inventory decides per option: port (used), park behind `transforms` (expressible), or deprecate loudly (unused); nothing dropped silently |
| Fixing the cross-cell cycles bug changes existing outputs | It is a correctness fix; called out in release notes with before/after example; xfail test documents the old behavior |
| Users with `autorun=True` muscle memory | kept, minus the I/O side effects — behavior is a strict subset |
| Collections saved by old code (hdf5, no meta) | `load_collection` reads bare frames with `meta=unknown`; only new saves get provenance |
| Sequencing deadlock with batch/plotting work | Only hard dependency is `batch.aggregate` (batch Phase 1.4); plotting handover is a separate later phase; curves fallback to `c.get_cap` until `cellpycore.curves` lands |

---

## 6. Open questions for the maintainer

1. **Package name:** `cellpy.collect` (proposed, verb-like, matches
   `cellpy.batch` / `cellpy.plotting`) — or fold into `cellpy.batch` as
   `batch.collect` since it always operates on a Batch?
2. **Where does the summary pipeline live** — `collect/summary.py`
   (proposed: it is collection-specific: rate filters, normalization,
   averaging) with `batch.aggregate` staying minimal (plain combine)?
   Or push grouping/normalization down into `batch.aggregate`?
3. **Default save format** parquet (proposed) — OK to demote hdf5 to
   legacy-writer status for collections?
4. **Provenance meta**: is `meta.json`-next-to-the-file enough, or should
   collections also be registrable in the journal/session (so a batch
   "knows" its derived products)?
5. `standard_gravimetric_collector`: keep the old callable name as a shim
   permanently (it is in real notebooks), or deprecate on the 2.1 cadence?
