# Plan: cellpy 2 suggested module layout

**Date:** 2026-07-16  
**Status:** planning sketch — not a migration schedule  
**Companion figure:** [`cellpy2-module-layout-planning.excalidraw`](cellpy2-module-layout-planning.excalidraw)

This document records the suggested package layout for cellpy 2: two packages,
a hexagonal seam, plain values only. Package homes are drawn from the existing
topic plans (batch / plotting / collect / instruments / cellpy_file). It does
not override those plans; it shows how they fit together on disk.

**How to read:** §1 layers → §2 `cellpy/` tree → §3 `cellpycore/` → §4 data flow
→ §5 today→target → §6 future developments (F1–F3).

---

## Legend (migration fate)

| Tag | Meaning |
|---|---|
| **keep** | Already present (or stays with a thin port) |
| **promote / new** | Lift to a new top-level package (or create) |
| **shim** | Compatibility surface: warn in 2.0, remove in 2.1 |
| **retire** | Emptied / deleted after the move |
| **future (F1–F3)** | Not scheduled; natural home marked for later work |

---

## 1. Layers (who owns what)

```text
┌──────────────────────────────────────────────────────────────────┐
│ USER API  (v1 patterns via shims)                                │
│   cellpy.get() · CellpyCell facade · batch / collect / filters / │
│   plotting · c.mass etc. → property facade · prms.* → config shim│
│   · c.schema                                                     │
├──────────────────────────────────────────────────────────────────┤
│ APP LAYER  — cellpy (content + policy)                           │
│   config/          pydantic-settings + TOML + override()         │
│   instruments/     parse() + harmonize() + entry-point registry  │
│   cellpy_file/     v9 parquet write · v8/legacy read → to_native │
│   batch/ collect/  orchestration · journals · multi-cell frames  │
│   plotting/        prepare → spec → render (single plot home)    │
│   MetaResolver     kwargs > journal/db > raw file > config       │
├────────────────── seam: plain values only ───────────────────────┤
│   frames, units, meta drafts — never config / files / pint Qty   │
├──────────────────────────────────────────────────────────────────┤
│ ENGINE  — cellpycore (shapes + tools)                            │
│   CellpyCellCore · Data (polars, native Cols, test_id)           │
│   make_step_table · make_summary · curves · merge · timestamps   │
│   metadata/ · units/ · legacy/                                   │
│   No file / env / network I/O · metadata optional on hot path    │
└──────────────────────────────────────────────────────────────────┘
```

Governing rule (from [`cellpy2-architecture-plan.md`](cellpy2-architecture-plan.md)):
core owns *shapes and tools*; cellpy owns *content and policy*. Everything
crossing the seam is plain values.

---

## 2. Suggested `cellpy/` package tree

```text
cellpy/
  __init__.py · cli.py · log.py · exceptions.py · _deprecation.py   [keep]
      thin public surface · warn_once · exceptions rooted in core

  config/                    [keep — Stage 1]
      models · loader · session · migrate

  parameters/                [shim]
      prms.* maps onto config (user notebooks; guts gone in 2.1)

  CellpyCell facade          [promote / shrink]
      today: readers/cellreader.py
      thin shell: load/save · delegate engines · property facade

  # --- I/O adapters (split today's readers/) ---
  instruments/               [promote from readers/instruments]     ← F2
      contract · registry · harmonize() · loaders

  cellpy_file/               [promote from readers/cellpy_file]
      format · read · write · translate (I/O only)

  db/  (+ filefinder)        [promote from db readers]              ← F1
      journal / lab meta feed for MetaResolver

  # --- Product packages (promote out of utils/) ---
  batch/                     [new]                                  ← F3 (layout)
      journal · policy · runner · store · layout · facade

  plotting/                  [new from plotutils]
      prepare → spec → render · figure registry

  collect/                   [new from collectors]
      Collection + provenance · plot() → plotting/

  filters/                   [keep]
      cycles · summary (mask → polars exprs)

  exporters/                 [keep / extend]                        ← F3
      entry-point contract · BDF reference · (future layered parquet)

  utils/                     [shim + slim remainder]
      helpers · ica · ocv_rlx · live/processor · example_data ·
      diagnostics
      + import shims: utils.batch / utils.plotutils / utils.collectors
      easyplot → gone at 2.0

  internals/                 [keep]
      OtherPath · connections (path/remote boundary)

  libs/                      [keep]
      vendored (fastnda, apipkg) — keep private
```

**Retire after the moves:** emptied `readers/`, `utils/batch_tools/`, `easyplot`,
and the guts of `parameters/` once the shim window ends.

### Types live next to their owner

Prefer not a grab-bag `base/` package:

| Type | Home |
|---|---|
| `LoaderResult` / `InstrumentLoader` | `instruments/contract.py` |
| `Data` / column schemas | `cellpycore` |
| `Journal` / `BatchResult` | `batch/` |
| Shared exceptions | `exceptions.py` (rooted in `cellpycore.exceptions`) |

---

## 3. `cellpycore/` package (PyPI pin)

```text
cellpycore/
  cell_core.py                 Data · CellpyCellCore · validate_raw
  summarizers.py · extractors.py   steps + summary engines
  curves.py                    get_cap family · CurveCols schema
  merge.py · timestamps.py     multi-test · epoch ns UTC
  metadata/                    models · io (scaffolding only)
  units/                       spec · converters
  legacy/                      mapping · headers bridge
```

Consumer truth is the PyPI pin in cellpy’s `[project.dependencies]`. Local
dual-repo work uses an editable overlay (see the migration guide in
`cellpy-core`).

---

## 4. Happy-path data flow

```text
vendor file
    → instruments (parse + harmonize)
    → MetaResolver + CellpyCell
    → cellpycore (steps / summary)
    → cellpy_file (save v9)
    → batch / collect / plotting
```

Translation (`to_native` / `to_legacy`) happens **once** at `cellpy_file` I/O —
never per engine call.

**Alt sources (future):** F2 can replace “vendor file”; F1 feeds MetaResolver;
F3 is an export branch off the processed data (not `cellpy_file`).

---

## 5. Today → target (`readers/` and `utils/`)

| Today | Target |
|---|---|
| `readers/cellreader.py` (7k+) | CellpyCell facade at package root |
| `readers/instruments/` | `instruments/` |
| `readers/cellpy_file/` | `cellpy_file/` |
| `readers/` dbreaders · filefinder · `data_structures.py` | `db/` (+ filefinder); structures relocate with owners |
| `utils/` (batch, plot, collect, …) | `batch/` · `plotting/` · `collect/` · slim `utils/` + shims |
| `parameters/` as source of truth | `config/` + `parameters/` shim |

### Owning plans

| Home | Plan |
|---|---|
| overall layers / seam | [`cellpy2-architecture-plan.md`](cellpy2-architecture-plan.md) §2–5 |
| `instruments/` | [`cellpy2-loader-port-and-extraction-plan.md`](cellpy2-loader-port-and-extraction-plan.md) |
| `cellpy_file/` | [`cellpy-file-loading-refactor-plan.md`](cellpy-file-loading-refactor-plan.md) |
| `batch/` | [`cellpy2-batch-redesign-plan.md`](cellpy2-batch-redesign-plan.md) |
| `plotting/` | [`cellpy2-plotting-redesign-plan.md`](cellpy2-plotting-redesign-plan.md) |
| `collect/` | [`cellpy2-collectors-redesign-plan.md`](cellpy2-collectors-redesign-plan.md) |
| `config/` + `parameters` shim | [`cellpy2-configuration-and-parameters-plan.md`](cellpy2-configuration-and-parameters-plan.md) |
| MetaResolver / meta | [`cellpy2-metadata-handling-plan.md`](cellpy2-metadata-handling-plan.md) |
| utils waves / filters / exporters | [`cellpy2-utils-migration-plan.md`](cellpy2-utils-migration-plan.md) |

---

## 6. Future developments — where they fit

Not scheduled yet. Markers show the natural home so later work does not invent
a parallel I/O stack.

### F1 — BatBase metadata (Django API + PostgreSQL)

Pull lab / cell / test metadata from BatBase.

| | |
|---|---|
| **Home** | `db/` — BatBase client adapter (HTTP and/or SQL) |
| **Also** | MetaResolver (APP LAYER) consumes drafts; `batch/` may seed journals from BatBase; `config/` holds URL + `SecretStr` credentials |
| **Not in** | `cellpycore` — only metadata *scaffolding* stays in core |

### F2 — Tester DB queries (pull cycling data)

A battery tester that exposes a queryable database for raw cycling data.

| | |
|---|---|
| **Home** | `instruments/` — an `InstrumentLoader` that queries the tester DB (same contract as file loaders: parse/query → `harmonize()` → `LoaderResult`) |
| **Also** | `config/` for DSN/secrets; later `utils/live` (or successor) for polling a running test |
| **Not in** | `db/` — that package is for lab/meta journals, not cycler raw |

Same “talk to a database” shape as F1, **different seam**: meta journals vs
instrument raw.

### F3 — Layered parquet export (raw / steps / cycles / tests)

Export four layers with a defined file-naming and folder-structure specification.

| | |
|---|---|
| **Home** | `exporters/` — entry-point exporter + layout spec (naming + folders for the four layers) |
| **Also** | `batch/layout.py` if the export is batch-scoped |
| **Not in** | `cellpy_file/` — that is the native v9 cellpy container (zip-of-parquet), a different product |
| **Core** | supplies the frames only; naming and folder policy stay in the app layer |

---

## 7. Iteration log

| Date | Change |
|---|---|
| 2026-07-16 | Initial sketch + Excalidraw companion; F1/F2/F3 future homes recorded |
