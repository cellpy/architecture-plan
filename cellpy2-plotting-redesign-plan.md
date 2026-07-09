# Plan: redesign of the plotting stack (plotutils as the single plotting home)

**Date:** 2026-07-09
**Closes:** the design gap for wave 2 of the
[utils migration plan](cellpy2-utils-migration-plan.md) ("plotutils: port,
slim" / "collectors: port"), and the plotting half of the
[batch redesign plan](cellpy2-batch-redesign-plan.md) §4.7 (batch delegates
`plot()` outward; `batch_plotters.py` is not ported).
**Direction (maintainer request):** plotutils hosts **all** plotting
capabilities; collectors *uses* plotutils under the hood and adds
collection/orchestration on top.
**Foundation:** [unit plan](unit-handling-cellpy2-plan.md) Phase 4
(`units_label`), [loader plan](cellpy2-loader-port-and-extraction-plan.md)
§2.3 (`cellpycore.curves`), [usage report](data-and-cellpycell-usage-in-cellpy-utils.md),
[conventions plan](cellpy2-conventions-plan.md) (deprecation policy).

---

## 1. Where plotting lives today

Four modules, ~12,800 lines, four overlapping generations:

| Module | Lines | Role | Redundancy with |
|---|---|---|---|
| `utils/plotutils.py` | 6 272 | Single-cell plots: `summary_plot` (new builder pipeline **and** its 1 400-line legacy twin), `raw_plot`, `cycle_info_plot`, `cycles_plot`; figure IO; plotly templates; color/marker helpers | collectors (fig IO, legend/marker helpers, templates) |
| `utils/collectors.py` | 3 046 | Batch-level collect-then-plot: `BatchCollector` + subclasses, `sequence_plotter` (~550 lines), `summary_plotter`, `cycles_plotter`, `spread_plot`; own fig IO; own templates | plotutils (everything visual), batch_plotters |
| `utils/batch_tools/batch_plotters.py` | 1 544 | Batch summary plots in **four** backends (bokeh 575 lines, matplotlib, plotly, seaborn), wired into the farm/barn machinery | plotutils + collectors summary plots |
| `utils/easyplot.py` | 1 953 | Older convenience layer; already verdicted **deprecate 2.0 / remove 2.1** (utils plan §2) | all of the above |

The same *figure* — "capacity and coulombic efficiency vs cycle" — can today
be produced by four different code paths that do not share a line of layout
logic.

### 1.1 Duplicated plumbing (verified copies)

- Figure save/load: `load_figure`, `load_plotly_figure`,
  `load_matplotlib_figure`, `save_matplotlib_figure`,
  `make_matplotlib_manager` exist in **both** `plotutils.py:103–166` and
  `collectors.py:54–121`.
- Legend/marker post-processing: `_plotly_legend_replacer` /
  `legend_replacer` and `_plotly_remove_markers` / `remove_markers` exist in
  **three** places (`plotutils.py:294–447`, `collectors.py:1723–1779`,
  `batch_plotters.py:879–914`).
- Plotly template construction: `_make_plotly_template` in
  `plotutils.py:245` and `batch_plotters.py:915`; collectors registers a
  *different* template family (`fig_pr_cell`, `fig_pr_cycle`, `film`,
  `summary`, `collectors.py:333–338`).
- Column-set selection for summary plots: `SummaryPlotInfo._create_col_info`
  (`plotutils.py:945–1122`, an ~180-line if/elif chain on magic strings),
  `_get_capacity_columns` (`batch_plotters.py:981`), and collectors' own
  column pickers.

### 1.2 The half-finished refactor inside plotutils

`summary_plot` already has the right skeleton: a `SummaryPlotConfig`
dataclass → `SummaryPlotInfo` → `SummaryPlotDataPreparer` →
`PlotlyPlotBuilder` / `SeabornPlotBuilder` (`plotutils.py:643–3202,
4644–4904`). But:

- `summary_plot_legacy` (`plotutils.py:3203–4643`, ~1 400 lines) still ships
  alongside it — two implementations of the same figure that must be
  bug-fixed in tandem (the issue #366 note at `plotutils.py:712–724` is
  exactly such a double-patch).
- `PlotlyPlotBuilder` hand-rolls axis-domain arithmetic per row count in
  four separate methods (`_configure_formation_{1,2,3,4}_rows`,
  `plotutils.py:1861–2149`) — a private faceting engine, ~500 lines, that
  breaks whenever a fifth row appears.
- `SeabornPlotBuilder` re-implements the same layout decisions for the
  static backend (`plotutils.py:2347–3202`) with its own per-family
  `_build_*_info_dicts` methods. Adding one new y-set means touching both
  builders plus the col-info chain.
- `summary_plot` exposes **40 parameters**, mirrored 1:1 as config fields
  with `from_kwargs`/`to_kwargs` round-trip machinery.

### 1.3 API inconsistencies (user-visible)

- Backend selection: `interactive: bool` (summary/raw/cycles/cycle_info) vs
  `backend: str` (collectors, `cycles_plot` internals). The seaborn backend
  exists only for `summary_plot`; matplotlib only for `raw_plot`/
  `cycle_info_plot`/`cycles_plot`; bokeh only in batch_plotters.
- `cycles_plot` accepts both `x_range`/`y_range` **and** `xlim`/`ylim`
  (`plotutils.py:5674–5677`) — two spellings for the same thing in one
  signature.
- Axis labels/units are assembled ad hoc per function
  (`_get_capacity_unit`, label dicts in `SummaryPlotInfo`, per-plotter
  `x_unit`/`y_unit` string args in collectors) instead of one labeling
  facility.
- ~150 lines of `_check_*` developer smoke functions live at the bottom of
  the module (`plotutils.py:6118–6272`).
- `create_colormarkerlist_for_journal` (`plotutils.py:552`) couples
  plotutils to the batch journal — the dependency should point the other
  way.

### 1.4 What is genuinely good (and must survive)

- The `y="capacities_gravimetric_coulombic_efficiency"` **named-figure
  idea**: one string buys a fully styled multi-panel figure. Users love it.
- Formation-cycle split panels, CV-share partitioning
  (`partition_summary_cv_steps`), rate filtering (issue #363 work) — real
  domain features no generic plotting library gives you.
- `return_data=True` returning the prepared frame.
- Collectors' *collection* half: gathering curves/ica/summaries across a
  batch with caching, autonaming and file output is a real workflow.

---

## 2. Design principles

1. **One home for figures.** Every figure cellpy can draw is produced by
   the plotting package. Collectors, batch, easyplot-successors and future
   dashboards call it; none of them draw.
2. **Prepare → spec → render.** Each plot family is (a) a data-preparation
   function producing a tidy long-format frame, (b) a declarative
   `FigureSpec` (panels, axes, labels, ranges, styling), and (c) a backend
   that renders specs. Layout is computed **once**, not per backend.
3. **A single cell is a batch of one.** Tidy frames carry optional
   `cell`/`group`/`sub_group` columns. The same `summary_plot` renders one
   cell or twenty; collectors stops needing its own plotters. This is the
   mechanism behind the maintainer's "collectors uses plotutils under the
   hood".
4. **Declarative figure registry.** The magic y-strings become entries in a
   data table (`PlotFamily` records), not an if/elif chain — listable,
   documented, and user-extensible (`register_family(...)` for the "added
   functionality" hook).
5. **Two backends, one switch.** `backend="plotly" | "matplotlib"`
   everywhere (`interactive=` kept as deprecated alias). Seaborn becomes
   styling inside the matplotlib backend, not a third backend. Bokeh goes
   with batch_plotters (batch plan §4.7).
6. **Labels come from the unit system.** All axis/legend text via
   `units_label()` (unit plan Phase 4) through one `labels.py`; the three
   copies of legend-conversion die.
7. **Frames in, figures out.** Plot functions accept tidy frames plus a
   small `CellContext` (name, units, meta) extracted by one adapter from a
   `CellpyCell`, a `Batch`/`BatchResult`, or cellpycore-native frames. No
   `c.`-poking inside builders — this is what makes the package work
   unchanged after the polars/native-headers flip.
8. **Keep the beloved surface.** `summary_plot(c, y="...")`,
   `cycles_plot(c)`, `raw_plot(c)`, `cycle_info_plot(c)` keep their names,
   defaults and figure look. Signatures stay; exotic parameters may move
   into the config object with deprecation shims.

---

## 3. Proposed architecture

New package `cellpy/plotting/` (import path decision in §7; the
`cellpy.utils.plotutils` name stays importable regardless):

```
cellpy/plotting/
    __init__.py       # public API: summary_plot, cycles_plot, raw_plot,
                      #             cycle_info_plot, ica_plot, save_figure, load_figure
    context.py        # CellContext / BatchContext adapters (the ONLY code
                      #   that touches CellpyCell / Batch / cellpycore objects)
    registry.py       # PlotFamily records for every named y-set + register_family()
    prepare/
        summary.py    # filters, rate rescaling, normalization, formation marking,
                      #   CV partitioning  -> tidy frame + FigureSpec
        curves.py     # voltage-capacity curves (uses cellpycore.curves), interpolation
        raw.py        # raw signals, downsampling
        steps.py      # step/cycle info frames
        ica.py        # dq/dv frames (delegates math to utils.ica / cellpycore)
    spec.py           # FigureSpec / PanelSpec / AxisSpec dataclasses
    backends/
        base.py       # Backend protocol: render(spec, frame) -> figure
        plotly.py     # includes generic formation/facet layout (replaces the
                      #   four _configure_formation_*_rows methods with one)
        mpl.py        # matplotlib + seaborn styling
    theme.py          # ONE template/palette module (plotly templates incl.
                      #   fig_pr_cell/fig_pr_cycle/film/summary; mpl styles;
                      #   colormarker cycles — journal-agnostic)
    labels.py         # units_label()-based axis text, legend prettifying (one copy)
    figures.py        # save/load/export figures (one copy; kaleido handling)
```

Target: **≤ 4 000 lines** for the package. Together with collectors
slimming to ~800 lines (§4), the plotting estate shrinks from ~12 800 to
~4 800 lines while gaining a backend-consistent feature set.

### 3.1 The flow, concretely

```python
def summary_plot(source, y="capacities_gravimetric_coulombic_efficiency",
                 backend="plotly", **options) -> Figure:
    ctx    = context.from_source(source)          # CellpyCell | Batch | frame
    family = registry.get(y)                      # declarative PlotFamily
    config = SummaryPlotConfig.from_options(family, **options)
    frame, spec = prepare.summary.prepare(ctx, family, config)  # tidy + spec
    return backends.get(backend).render(spec, frame)
```

- `frame` is long-format with `cycle`, `value`, `variable`, `panel`,
  optional `cell`/`group` columns; `return_data=True` returns it (as today).
- `spec` declares panels/axes/ranges/labels once; `backends.plotly` and
  `backends.mpl` translate it. A new y-set or an extra panel row is a
  registry/prepare change only — the builders never fork per row count
  again.
- With a `cell` column present, the same spec grows per-cell coloring or
  faceting — which **is** collectors' `summary_plotter`/`sequence_plotter`
  functionality, now living in one place.

### 3.2 The registry (what replaces the magic strings)

```python
@dataclass(frozen=True)
class PlotFamily:
    name: str                      # "capacities_gravimetric_coulombic_efficiency"
    panels: tuple[PanelDef, ...]   # rows: columns to plot, per-panel kind/range keys
    mode: str = "gravimetric"      # unit mode driving units_label()
    supports_formation: bool = True
    supports_cv_split: bool = False
    extras: dict = ...             # e.g. rate panel, normalization defaults
```

All 15 current y-sets become records; `plotting.families()` lists them with
descriptions (better error message than today's silent fallthrough);
`register_family()` lets power users add their own set without touching
cellpy — the requested "added functionality" without new framework.

### 3.3 Collectors on top (the layering the maintainer asked for)

`collectors.py` keeps its name and its class, loses its drawing code:

```
BatchCollector
    ├── collect:  gather frames across b.cells (cellpycore.curves / summaries)
    │             + caching, csv/parquet persistence          [KEEPS]
    ├── name/paths: autonaming, output folders                [KEEPS]
    ├── plot:     calls cellpy.plotting.* with the collected  [DELEGATES]
    │             frame (which already carries cell/group columns)
    └── save:     figures via plotting.figures                [DELEGATES]
```

- `sequence_plotter`, `summary_plotter`, `cycles_plotter`, `spread_plot`,
  the duplicate fig IO and the local templates are removed; their
  capabilities (fig-per-cell vs fig-per-cycle faceting, film/heatmap mode,
  spread bands) become options of `plotting.plots.curves` (`layout=
  "per_cell" | "per_cycle"`, `kind="line" | "film" | "spread"`).
- The argument-elevation machinery collapses to two explicit dicts:
  `collector_options` and `plot_options` (passed straight to plotting).
- `BatchSummaryCollector` gains the native-only columns opt-in (utils plan
  F6) purely through the registry — no new plotting code.
- Batch facade `b.plot(...)` (batch plan §4.6) calls
  `plotting.summary_plot(b, ...)` directly; the collector adds value only
  when you want persistence/caching of the collected frames.

### 3.4 What dies

| Today | Fate |
|---|---|
| `summary_plot_legacy` + builder duplication | deleted after parity gate (§5 Phase 2) |
| `batch_plotters.py` (incl. bokeh) | not ported (batch plan §4.7) |
| `easyplot.py` | deprecated 2.0 → removed 2.1 (already decided) |
| duplicate fig IO / legend helpers / templates | single copies in `figures.py` / `labels.py` / `theme.py` |
| `_check_*` dev functions in module | move to `dev/` smoke notebook + tests |
| `interactive=` bool, `xlim`/`ylim` dupes | deprecated aliases for one minor release |
| `create_colormarkerlist_for_journal` | journal-agnostic `theme.color_marker_cycle(groups)`; batch passes groups |

---

## 4. Sizing

| Piece | Today | After |
|---|---|---|
| plotutils | 6 272 | ~3 200 (package) |
| collectors | 3 046 | ~800 |
| batch_plotters | 1 544 | 0 |
| easyplot | 1 953 | 0 (per existing verdict) |
| **total** | **12 815** | **~4 000–4 800** |

---

## 5. Migration plan

Wave-2 work in the utils plan's sequencing (needs `units_label` from unit
plan Phase 4 — pull it forward if late — and `cellpycore.curves` for the
collectors data side). Phases are shippable increments; the old code keeps
working until its replacement passes the gates.

### Phase 0 — golden figures (½–1 day)

1. A script/notebook that renders the full figure menu (every registered
   y-set × both backends × single-cell and batch input) from the test cells
   in `cellpy-core-dev/data`, exporting HTML/PNG. The existing marimo
   validation notebooks (`cellpy-core-dev/notebooks/`) are the natural
   harness — add a `07_plot_gallery.py` that becomes the visual regression
   deck.
2. Structural assertions (not pixel diffs): trace count, panel count, axis
   titles, first/last data values per trace. These become the parity gate.
3. Inventory which plot parameters are actually exercised in docs,
   tutorials and the maintainers' own notebooks → the must-keep list.

### Phase 1 — consolidate foundations (1–2 days)

Create `figures.py`, `theme.py`, `labels.py` with the **single** copies;
old locations (`plotutils`, `collectors`) import from them with
deprecation-free shims (pure moves). Registry (`registry.py`) created by
mechanically translating `_create_col_info` — behavior identical, chain
deleted. No figure changes; golden set must be byte-stable for plotly JSON.

### Phase 2 — spec + backends, summary first (4–5 days)

1. `spec.py` + `backends/plotly.py`: one generic panel/formation layout
   engine replacing the four `_configure_formation_*_rows` methods.
2. Port `prepare/summary.py` from `SummaryPlotDataPreparer` (mostly reuse —
   this part of the existing refactor is sound).
3. `summary_plot` switches to the new path; **parity gate**: golden
   structural assertions + `return_data` frame equality vs the old
   pipeline, both backends.
4. `backends/mpl.py` renders the same specs; `SeabornPlotBuilder` and
   `summary_plot_legacy` are deleted in the same PR that proves parity
   (escape hatch: `CELLPY_LEGACY_SUMMARY_PLOT=1` env flag for one minor
   release).

### Phase 3 — the other families (3–4 days)

`cycles_plot`, `raw_plot`, `cycle_info_plot`, plus an `ica_plot` family
(new, cheap on this architecture) ported to prepare/spec/render. The
curve-family preparation absorbs collectors' `cycles_collector` reshaping
and targets `cellpycore.curves` output directly. Backend parity per family
via the golden set.

### Phase 4 — collectors re-based (2–3 days)

Collectors' plotters deleted; `BatchCollector.plot` delegates per §3.3;
templates unified into `theme.py`; batch facade `plot()` wired. Golden set
gains the batch-input column.

### Phase 5 — removals (on the 2.x cadence)

`easyplot` removal (2.1, already scheduled), `batch_plotters` removal with
the batch redesign Phase 4, deprecated aliases (`interactive=`,
`xlim`/`ylim`, old import paths) removed per the conventions plan.

Total: **~2–2.5 weeks**, sequenced after batch redesign Phases 0–2 (the
tidy `summaries` frame from `aggregate.py` is the batch-side input
contract).

---

## 6. Risks

| Risk | Mitigation |
|---|---|
| Visual regressions that structural assertions miss (spacing, colors) | Phase-0 gallery reviewed by eye at each phase gate; plotly JSON snapshots catch styling drift cheaply |
| The 40-parameter `summary_plot` surface breaks someone | Signature preserved verbatim; params map into config; unknown-kwarg warnings name the replacement |
| Seaborn look loyalists | `backend="matplotlib"` ships the seaborn *styling* (palette/style kwargs kept); document the equivalence |
| `cellpycore.curves` not landed when Phase 3 starts | prepare/curves takes frames; the adapter falls back to `c.get_cap` until curves lands (same trick as the validation notebooks) |
| Registry can't express some exotic current figure | `PlotFamily.extras` escape hatch + keep the old function importable until its family passes the gallery gate |
| Two design efforts (batch + plotting) colliding in collectors | Explicit contract: batch/aggregate produce tidy frames (batch plan §4.5); plotting consumes frames; collectors orchestrates — interfaces before internals |

---

## 7. Open questions for the maintainer

1. **Package name/path:** `cellpy.plotting` (proposed, mirrors
   `cellpy.batch`) with `cellpy.utils.plotutils` kept as permanent re-export
   — or keep `plotutils` as the real home?
2. **Bokeh confirmed dead?** (Also asked in the batch plan; this plan
   assumes yes.)
3. **Seaborn as styling-only** inside the matplotlib backend — acceptable?
   (Today it is the *only* static backend for `summary_plot`, so static
   output changes engine from seaborn objects to plain matplotlib.)
4. Should `register_family` be public API in 2.0 (documented, semver-bound)
   or provisional (`_register_family`) for one release?
5. `cycle_info_plot`'s matplotlib path supports single cycles only — unify
   to multi-cycle in the port (behavior change) or keep the asymmetry?
