# Plan: the total cellpy 2 architecture (MVP → complete)

**Date:** 2026-07-09 (living document — iterate here; see iteration log at the end)
**Status:** draft v2 — updated after the
[gap analysis](cellpy2-plans-gap-analysis.md) and the six plans it spawned.
This document *coordinates* the topic plans; it decides nothing they own.
**Scope:** The overall shape of the cellpy 2 solution: package topology, layers, the
MVP cut vs. the complete system, the design patterns we commit to, the formal
instrument-loader contract, the implementation timeline, and the safe-to-use API
surfaces for util and loader developers.

---

## 0. The plan set (what owns what)

| Plan document | Owns |
|---|---|
| [cellpy2-configuration-and-parameters-plan.md](cellpy2-configuration-and-parameters-plan.md) | Config stack (pydantic-settings/TOML), prms shim, secrets, OtherPath decision (§5b) |
| [cellpy2-metadata-handling-plan.md](cellpy2-metadata-handling-plan.md) | `CellMeta`/`TestMetaCollection` wiring, meta mapping, `MetaResolver`, meta persistence |
| [unit-handling-cellpy2-plan.md](unit-handling-cellpy2-plan.md) | One unit spec/registry, converter delegation, `units_label`, unit policy |
| [cellpy-file-loading-refactor-plan.md](cellpy-file-loading-refactor-plan.md) | Extracting cellpy-file I/O out of `CellpyCell` (behavior-preserving, v8) |
| [cellpy2-native-headers-migration-plan.md](cellpy2-native-headers-migration-plan.md) | Native `Cols` end-to-end, `translate.py`, v9 format, the flip oracle, accessor shim |
| [cellpy2-polars-port-execution-plan.md](cellpy2-polars-port-execution-plan.md) | pandas→polars on the cellpy side; keys-in-columns; the flip's frame-type half |
| [cellpy2-loader-port-and-extraction-plan.md](cellpy2-loader-port-and-extraction-plan.md) | Loader port to harmonized raw (`harmonize()` + declarations), `cellpycore.curves` |
| [cellpy2-utils-migration-plan.md](cellpy2-utils-migration-plan.md) | Utils/batch/plotting/filters migration in waves; the "utils contract"; live/incremental |
| [cellpy2-batch-redesign-plan.md](cellpy2-batch-redesign-plan.md) | Batch v3 architecture (journal/policy/runner/result/store) and the migration off farm/barn `batch_tools` |
| [cellpy2-plotting-redesign-plan.md](cellpy2-plotting-redesign-plan.md) | One plotting home (prepare→spec→render, figure registry); collectors re-based on it; batch_plotters/easyplot retired |
| [cellpy2-collectors-redesign-plan.md](cellpy2-collectors-redesign-plan.md) | Collection layer: `Collection` with provenance, options dataclasses replacing "elevated arguments", `concat_summaries` consolidation |
| [cellpy2-ica-redesign-plan.md](cellpy2-ica-redesign-plan.md) | ICA + DVA: `dqdv()` and new `dvdq()` on one pure core, `IcaOptions`, specced long-format output frames |
| [cellpy2-conventions-plan.md](cellpy2-conventions-plan.md) | Exceptions, logging, deprecation cadence (`warn_once`, `DEPRECATIONS.md`) |
| [cellpy2-release-and-branching-plan.md](cellpy2-release-and-branching-plan.md) | 2.0 support matrix, trunk-based flip, cross-repo merge order, benchmarks |
| [cellpy2-plans-gap-analysis.md](cellpy2-plans-gap-analysis.md) | The cross-read that produced the set; ownership table in its §3 |

**Builds on (cellpy-core design docs):**
`cellpy-core/.issueflows/04-designs-and-guides/cellpy-core-migration.md`,
`cellpy-core-integration-roadmap.md` (STEPs 01–12), `cellpy-core-integration-into-cellpy.md`.

---

## 1. The total solution in one sentence

Cellpy 2 is a **layered, two-package system**: `cellpycore` is a small, pure,
polars-based compute engine that owns *shapes and tools* (schemas, step/summary/curve
engines, metadata models, unit converters, the legacy mapping), and `cellpy` becomes a
thin application layer that owns *content and policy* (configuration, instrument
loaders, metadata population, persistence, plotting, batch). Everything crossing the
seam is plain values — never config objects, pint quantities, or file handles.
Translation between the v1 dialect and the native schema happens **once, at I/O
boundaries**, not per call. The v1 API survives through deprecation shims on the
shared cadence (introduced 2.0, removed 2.1).

**Where we already are:** the compute engine (steps, summary, headers, timestamps,
metadata scaffolding, unit tooling incl. `cellpycore/units/spec.py`) lives in
`cellpycore` behind the `OldCellpyCellCore` bridge, guarded by contract tests and
golden fixtures (integration roadmap STEPs 01–11 ✅). The gap analysis confirmed the
planning surface is now **complete**: every package in cellpy has an owning plan
(utils wave 0 closes the last scanning blind spot: `filters/`, `exporters/`,
`internals/` consumers).

---

## 2. MVP cellpy 2 (the 2.0 release)

**Redefined 2026-07-09.** The release/branching plan fixed the 2.0 support matrix:
cellpy 2.0 ships **polars frames, native headers, and v9 files** — the flip is *in*
the MVP, not after it. (The previous draft's "keep v8 + legacy dialect" MVP is now
the *preparation stage*: everything behavior-preserving lands on master v1.x-safe
before the flip; see the timeline, §6.) What keeps 2.0 minimal is the shim strategy —
every documented v1 pattern still works, warning once — and the wave structure:
only utils waves 1–2 (batch, helpers, filters, plotting) are 2.0-blocking.

```text
┌──────────────────────────────────────────────────────────────────┐
│ USER API (v1 patterns work via shims, warn once → DEPRECATIONS.md)│
│   cellpy.get() · CellpyCell facade · batch/helpers/filters/      │
│   plotutils/collectors (utils waves 1–2) · cellpy convert CLI    │
│   c.mass etc. → property facade · prms.* → config shim           │
│   c.schema = public header API (headers_* become the shim)       │
├──────────────────────────────────────────────────────────────────┤
│ APP LAYER (cellpy 2)                                             │
│   config/       pydantic-settings + TOML + platformdirs;         │
│                 override() ctx-mgr; provenance; setup migrate    │
│   MetaResolver  kwargs > journal/db > raw file > config defaults │
│   cellpy_file/  v9 parquet native (write) · v8/legacy read →     │
│                 to_native() once · save(format="v8") compat      │
│   instruments/  vendor parse() + declaration-driven harmonize(); │
│                 tier-1/2 loaders emit harmonized raw + drafts    │
│   conventions   exception tree rooted in core · opt-in logging · │
│                 warn_once helper · secrets env-only (SecretStr)  │
├────────────────── seam: plain values only ───────────────────────┤
│ ENGINE (cellpycore, PyPI pin) — native path                      │
│   CellpyCellCore · Data (polars, native Cols, test_id keys)      │
│   make_step_table · make_summary · curves · merge · timestamps   │
│   metadata models/io · units spec+converters · legacy/mapping    │
│   (OldCellpyCellCore serves only the v1.x maintenance branch)    │
└──────────────────────────────────────────────────────────────────┘
    oracle: value parity through the mapping on the golden cells
    guard:  benchmarks vs the v1.x baseline — no metric slower
```

2.0 definition of done (from the release plan, restated): reads v8+v9, writes v9
(+ `save(format="v8")`), v<8 handled by `cellpy convert` on 1.x; pytables demoted to
a `legacy-files` extra; dependency deltas per release plan §5; benchmark acceptance;
every shim registered in `DEPRECATIONS.md`.

---

## 3. Complete cellpy 2 (2.x)

What the complete system adds over the 2.0 MVP:

```text
┌──────────────────────────────────────────────────────────────────┐
│ USER API                                                         │
│   canonical access only (shims removed in 2.1):                  │
│   c.data.meta.mass · c.data.tests[tid].… · c.schema.raw.potential│
│   units_label() / with_cellpy_unit() presentation helpers        │
│   live/incremental: poll a running test, refresh steps/summary   │
│   batch reports with test-level summaries + native-only columns  │
│   (energies, powers, durations) · exclude_step_types on summaries│
├──────────────────────────────────────────────────────────────────┤
│ APP LAYER                                                        │
│   utils waves 3–4 done: ica + ocv_rlx on cellpycore.curves;      │
│   live.py/processor.py rebuilt on core incremental (or dropped   │
│   deliberately) · easyplot removed · docs/CLI inventory done     │
│   corrected IR semantics as default (documented oracle exception)│
├──────────────────────────────────────────────────────────────────┤
│ IO / PLUGIN LAYER                                                │
│   entry-point loader registry incl. third-party loaders          │
│   tier-3 decisions executed (biologics/batmo ported; nda parked) │
│   BDF adapter placement decided · JSON-LD/DB transport           │
│   implementable against core's io stubs (TestMeta.uuid linkable) │
├────────────────── seam: plain values only ───────────────────────┤
│ ENGINE (cellpycore)                                              │
│   volumetric mode implemented · test-level summaries             │
│   SPEED-30 headers (value+unit+dtype, versioned) slotted behind  │
│   the Schema indirection — resolves per-column units for good    │
│   OldCellpyCellCore + shims deleted when v1.x support ends       │
└──────────────────────────────────────────────────────────────────┘
```

Deliberately deferred with a parking spot (not forgotten): cell-identity UUIDs +
BattINFO/EMMO ontology mapping (metadata plan open question 6, F10), fsspec/remote
schemes beyond ssh (config plan §5b revisit trigger), narwhals (polars plan
decision 2 revisit trigger), per-test `raw_units` for mixed-instrument merges
(metadata plan open question 6).

---

## 4. Design patterns we commit to

| Pattern | Where it applies in cellpy 2 |
|---|---|
| **Hexagonal / ports-and-adapters** | The governing decision — "core owns shapes and tools; consumer owns content and policy" *is* this pattern. Core is the pure domain (no file/env/network I/O, enforced by CI lint); loaders, persistence, and db are adapters. |
| **Functional core, imperative shell** | Engines are pure functions over frames; all side effects (file reads, config resolution) happen in the shell and results cross the seam **by value** (the factors-by-value unit seam). Dependency injection done the pythonic way — pass values, not containers. |
| **Facade** | `CellpyCell` shrinks from a 7,400-line god object to a thin facade delegating to core engines, `cellpy_file/`, and `cellpycore.curves`. |
| **Adapter / anti-corruption layer, at I/O only** | `translate.py` (`to_native`/`to_legacy`) over `cellpycore.legacy.mapping`: translation happens **once at file boundaries**, reversing today's per-call rename sandwich (native-headers plan D1). The legacy HDF5 import shim and `save(format="v8")` are the same pattern. Deliberately disposable. |
| **Repository** | Stateless `cellpy_file/` package (format registry, `read/legacy_read/write/translate`) replacing hidden-mutable-state extraction methods. Core ships interface stubs; cellpy implements them. |
| **Strategy + plugin registry** | Instrument loaders behind one formal contract (§5), discovered via `importlib.metadata` entry points; built-ins use the same registry. |
| **Declarative configuration over code** | Loader normalization is driven by *declarations* (column_map, units, tz, reset granularity, aux_map) validated at registration time — a typo'd declaration fails at import, not mid-load (loader plan §2.2). |
| **Pipeline (two-stage ingestion)** | `vendor file → parse() → vendor frame + declarations → harmonize() → native raw`. The hard-won vendor parsing stays per-loader; everything after it is one shared, framework-owned stage (loader plan §2.1). |
| **Chain of responsibility (layered resolution)** | `MetaResolver` precedence (kwargs > journal/db > raw file > config defaults) and pydantic-settings source layering — both with inspectable provenance. |
| **Null object / graceful degradation** | Engines run with empty `CellMeta`/`TestMeta`; populated metadata is never required on the hot path (guarded by tests). |
| **Lazy singleton, scoped mutation** | Config as a lazily-created module attribute (PEP 562 `__getattr__`), never import-time I/O — the same rule now extended to logging (conventions plan §2: `NullHandler`, explicit `cellpy.log.setup()`). `override()` + pytest fixture end global-state test leaks. One memoized pint registry per process. |
| **Typed models at the boundary** | Plain dataclasses in core; pydantic v2 (`validate_assignment`, `SecretStr`) only in the app layer — validation fails loudly at load time. Fail-loudly posture codified in the conventions plan §4 (exactly two warn-only escape hatches: `local_instrument`, `accept_old`). |
| **Deprecation facade, one cadence** | All shims follow the conventions plan §3: introduced 2.0, removed 2.1, `warn_once` with the replacement named, self-registered in a generated `DEPRECATIONS.md`. Private `_` API breaks without deprecation. |
| **Golden master + contract tests** | Characterization before touching; field-by-field parity contracts; golden fixtures regenerated by script, never by hand (shared convention F8). **The oracle changes character at the flip**: byte-parity → value parity through the mapping, with documented exceptions for frozen legacy bugs (the IR semantics case, F4). |
| **Structural typing (`typing.Protocol`)** | The loader contract (§5) is a Protocol, not an ABC — third-party loaders need no cellpy base class; conformance is structural, checked in the registry and by the test-kit. |
| **Immutable-by-convention frames** | Engines return new frames rather than mutating in place; *keys live in columns, never in an index* is law (polars plan decision 3), lint-enforced (polars plan Phase D). |
| **Single exception hierarchy, rooted in core** | `cellpycore.exceptions.CellpyError` at the root; the app derives `ConfigurationError`, `UnitsError`, `LoaderError`, `CorruptCellpyFile`, … No bare `Exception`, no blanket `except Exception` at boundaries (conventions plan §1). |

---

## 5. The formal instrument-loader contract

Updated 2026-07-09 to align with the
[loader-port plan](cellpy2-loader-port-and-extraction-plan.md)'s two-stage design.
The Protocol below is the **outer** contract (what the registry and framework see);
the loader plan's `parse()` + `harmonize()` split is the **sanctioned implementation**
of it for built-in loaders.

### 5.1 Design decisions

1. **`typing.Protocol`, not an ABC.** Third-party loaders conform structurally; no
   inheritance, no import of a cellpy base class required. `@runtime_checkable` so the
   registry can `isinstance`-check at registration (the test-kit does the real
   conformance check).
2. **Two-stage implementation (loader plan §2.1).** Loaders keep their vendor-specific
   `parse()` (the hard-won part) and *declare* the rest — column_map, units, tz rule,
   reset-granularity per cumulative column, aux_map — in their configuration. The
   framework-owned `harmonize()` performs rename → cast (`RawCols.dtype_map()`) →
   timestamp conversion → reset-granularity normalization (core #42) → identity/
   provenance stamping → `validate_raw_frame` (strict; warn-only for
   `local_instrument`). Built-in loaders therefore implement `load()` as
   `harmonize(parse(source), declarations)`; third-party loaders may do the same or
   emit native frames directly — the contract only judges the result.
3. **The result is a frozen dataclass (`LoaderResult`),** never attribute-poking on
   `Data`. Loaders fill **only what the file actually knows**; provenance
   (`source_kind/type/uri`, `source_uuid`, `raw_file_names`, `loaded_datetime`,
   `uuid`, `test_id`, `mask`) is stamped by the framework.
4. **The raw frame is polars in the harmonized-raw schema** (`config.RawCols`:
   `potential` not voltage, `test_time` in seconds, `epoch_time_utc` int64 ns UTC,
   `datapoint_num` as a *column*). During the transition, legacy pandas loaders run
   behind a framework-side `LegacyLoaderAdapter`; the adapter dies at loader-plan
   Step 6.
5. **Units are declared, validated, and mandatory.** `raw_units` is a
   `cellpycore.units.CellpyUnits` (pint-parsable labels, checked by
   `validate_units()`) — never an ad-hoc dict, never float scale factors.
6. **Metadata is drafts.** `test_meta` is a partial `TestMeta`; `cell_meta` an
   optional partial `CellMeta`. Population policy belongs to `MetaResolver`.
7. **Capability metadata is class-level** (`name`, `instrument`,
   `supported_suffixes`) so the registry routes without instantiating.
8. **Sources are path-likes the framework resolves** (config plan §5b): OtherPath /
   remote handling is centralized in one boundary function; loaders receive a local
   `Path` and never encode ssh/scheme semantics (keeps the fsspec door open).

### 5.2 The contract (target home: `cellpy/instruments/contract.py`)

```python
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import ClassVar, Protocol, runtime_checkable

import polars as pl

from cellpycore.metadata.models import CellMeta, TestMeta
from cellpycore.units import CellpyUnits


@dataclass(frozen=True, slots=True)
class LoaderResult:
    """What an instrument loader hands back to the loading framework.

    The frame must conform to the harmonized-raw schema (config.RawCols,
    epoch_time_utc as int64 ns UTC, datapoint_num as a column — never an
    index). Meta objects are *drafts*: only fields the source file actually
    knows are filled. Provenance (source_*, raw_file_names, loaded_datetime,
    uuid, test_id, mask) is stamped by the framework.
    """

    raw: pl.DataFrame
    raw_units: CellpyUnits
    test_meta: TestMeta
    cell_meta: CellMeta | None = None
    warnings: tuple[str, ...] = field(default=())


@runtime_checkable
class InstrumentLoader(Protocol):
    """Structural contract for instrument loaders (cellpy 2).

    Implementations do NOT need to import or inherit anything from cellpy;
    conformance is structural. Register via the ``cellpy.loaders`` entry-point
    group. Loaders must be stateless across calls. Built-in loaders implement
    load() as harmonize(parse(source), declarations) — see the loader-port
    plan §2.1; only the LoaderResult is judged.
    """

    # -- capability metadata (class-level; registry routes on these) --------
    name: ClassVar[str]                        # e.g. "arbin_res"
    instrument: ClassVar[str]                  # e.g. "arbin"
    supported_suffixes: ClassVar[tuple[str, ...]]  # e.g. (".res",)

    # -- optional cheap pre-check -------------------------------------------
    def can_load(self, source: Path) -> bool:
        """Cheap sniff (suffix/magic bytes). Must not fully parse the file."""
        ...

    # -- the one required operation -----------------------------------------
    def load(
        self,
        source: Path,                             # local; framework resolves
                                                  # OtherPath/remote first (§5.1.8)
        *,
        instrument_config: object | None = None,  # typed per-instrument model
                                                  # from cellpy.config.instruments
        **kwargs: object,                         # loader-specific knobs, must be
                                                  # ignorable (lenient) when unknown
    ) -> LoaderResult:
        """Parse one source into a LoaderResult.

        Must raise LoaderError (wrapping vendor parse errors) on failure —
        never return a partial result silently (conventions plan §1).
        """
        ...
```

### 5.3 Registration and discovery

```toml
# in a loader package's pyproject.toml (works for cellpy's own built-ins too)
[project.entry-points."cellpy.loaders"]
arbin_res = "cellpy.instruments.arbin_res:ArbinResLoader"
neware_xlsx = "cellpy.instruments.neware_xlsx:NewareXlsxLoader"
mycycler = "cellpy_mycycler.loader:MyCyclerLoader"   # third-party
```

The registry (`cellpy/instruments/registry.py`) loads the `cellpy.loaders`
entry-point group lazily on first use, `isinstance`-checks against
`InstrumentLoader`, **validates each loader's declarations at registration time**
(loader plan §2.2 — a typo'd declaration fails at import of the configuration), and
routes by explicit `instrument=` argument first, then by `supported_suffixes` +
`can_load()`. Built-ins register through the same mechanism — no privileged path.

### 5.4 Framework responsibilities (not the loader's)

- Resolve the source: OtherPath/remote → local temp copy via the single boundary
  function in `cellpy_file/` (§5.1.8).
- Run `harmonize()` for declaration-based loaders; run `validate_raw_frame` strict on
  every load.
- Stamp provenance on the draft `TestMeta` (`source_kind=FILE`, `source_type`,
  `source_uri`, `source_uuid` = stable file-identity hash, `raw_file_names`,
  `loaded_datetime=now`, `uuid` = uuid4 at first load, preserved through
  save/load/merge).
- Assign `test_id` (and `mask=True`) into the raw frame rows.
- Run `MetaResolver` over (user kwargs, journal/db, drafts, config
  `ScienceDefaults`) to produce final `CellMeta` / `TestMetaCollection`.
- Convert meta quantities to cellpy units **once, at ingestion**.
- During transition: wrap legacy pandas loaders in `LegacyLoaderAdapter`
  (frame + header conversion via the mapping; unit-dict → `CellpyUnits`).

### 5.5 Conformance test-kit (ships with cellpy)

`cellpy.instruments.testing.check_loader(loader, fixture)` asserts:

1. `LoaderResult` types exact (polars frame, `CellpyUnits`, `TestMeta`).
2. The frame passes the harmonized-raw schema check (columns ⊆ `RawCols`, dtypes per
   `dtype_map()`, int64-ns timestamps, monotone `datapoint_num`, no index promotion).
3. `raw_units` validates (every label pint-parsable).
4. The draft `TestMeta` contains **no** provenance fields (a filled `source_uri` in a
   draft is a contract violation).
5. `can_load()` is cheap (time/IO budget on a large dummy file).
6. Determinism: two `load()` calls on the same file give equal results.
7. **Reset-granularity property test** (loader plan §5 — the correctness landmine):
   per-cycle capacities recomputed from the normalized raw equal the legacy pipeline's
   summary capacities on the loader's golden fixture.

Fixture convention (F8): committed small vendor file + expected harmonized parquet,
regenerated by script only, never by hand.

### 5.6 Open questions on the contract (iterate here)

- **Multi-test sources** (formats carrying several tests per file): recommend `load()`
  always returns `tuple[LoaderResult, ...]` (uniform) over `LoaderResult | list`.
  Decide before freezing the Protocol — still open.
- **Streaming/chunked loading** for very large files and the live/incremental path
  (utils plan wave 4): a second, optional protocol (`SupportsIncrementalLoad`) rather
  than complicating the base contract. Design it together with wave 4, since core's
  incremental summarization is the consumer.
- **`instrument_config` typing:** `object` now; tighten once the config plan's typed
  per-instrument models land.
- ~~OtherPath in the signature~~ — **resolved** (config plan §5b): loaders get a plain
  local `Path`; the framework resolves remote sources at one seam.

---

## 6. Timeline — which plan, in which order

The release plan's branch strategy shapes everything: **trunk-based**, all
behavior-preserving preparation on master (v1.x stays green), the two genuine
flag-days (native-headers Phase 3 + polars Phase C) combined into **one short-lived
flip branch** (>4 weeks alive = stop and re-slice), and `v1.x` forking **at** the
flip. Cross-repo rule (F9): core-first for additions (core PR → PyPI release →
cellpy re-pin), cellpy-first for removals.

### Stage 0 — foundations (days; do first, everything imports them)

| # | Work | Plan |
|---|---|---|
| 0.1 | Conventions PR: exception tree, `warn_once` + `DEPRECATIONS.md`, logging opt-in | conventions (whole plan) |
| 0.2 | Benchmark harness + **v1.x baseline captured before any polars work** | release §4 |
| 0.3 | Utils wave 0: scan `filters/`, `exporters/`, `internals/` consumers | utils §3 wave 0 |
| 0.4 | One-hour doc-sync pass in cellpy-core (stale STEP-12/#54 statuses — F7) | gap analysis F7 |

### Stage 1 — v1.x-safe preparation on master (parallel tracks, ~4–6 weeks)

| Track | Sequence | Plans |
|---|---|---|
| A — config | Steps 0–2 (characterization, constants purge, parallel stack) → 3 (shim swap) → 4 (call-site migration, kill import-time init) | config |
| B — files/headers | file-loading Steps 0–5 → native-headers Phase 1 (`translate.py`, dormant) → Phase 2 (v9 format + `cellpy convert`; **needs the container-layout decision**, shared with metadata Step 4) | file-loading, native-headers |
| C — frame hygiene | hardcoded-literals priorities 1–3 → polars Phase A (de-index raw/summary/journal) | native-headers Phase 0, polars |
| D — core seam (core-first merge order) | metadata Step 1 (mapping) → Step 2 (Data wiring, `test_id` composite keys) · units Phases 1–2 (one registry, delegate converters; `convert_value`/`calculate_scaler` added to core) · curves spec + `cellpycore.curves` (loader Step 2) → **core release → cellpy re-pin** | metadata, units, loader |
| E — loaders | loader Steps 0–1 (fixtures; `harmonize()` + declaration schema + pilot `maccor_txt`) → Steps 3–4 (tier 1, tier 2) as they become ready | loader (absorbs units Phase 3 + metadata Step 3) |

Gate for Stage 2: tracks B, C, D complete; track E at least through the pilot;
full v1.x suite green throughout.

### Stage 2 — the flip (one short-lived branch)

Native-headers Phase 3 + polars Phase C + loader Step 6 (delete legacy
normalization) in one branch: `CellpyCell.core` → lean `CellpyCellCore`, frames go
native-named **and** polars in the same migration, rename sandwich deleted, oracle
switches to value-parity-through-the-mapping (with the documented IR-semantics
exception, F4). `v1.x` maintenance branch forks here. Benchmarks must show the
bridge-removal win, not a regression.

### Stage 3 — 2.0 assembly (post-flip on master)

| # | Work | Plan |
|---|---|---|
| 3.1 | Native-headers Phase 4: `c.schema` public, `headers_*` → attribute shim; step-type literals → `StepType` enums | native-headers |
| 3.2 | Utils wave 1 (batch, helpers, filters, diagnostics — defines the **utils contract**) → wave 2 (plotutils, collectors; needs `units_label`, pull it forward if Phase 4 slips) | utils, units Phase 4 |
| 3.3 | Metadata Steps 3–6: loader drafts everywhere, `MetaResolver`, v9 meta persistence, merge, journal consumers | metadata |
| 3.4 | Config Steps 5–7: `cellpy setup` writes/migrates TOML; secrets hardening; generated docs | config |
| 3.5 | Loader Step 5: tier-3 decisions (port biologics/batmo, park ext_nda, warn-only local_instrument) | loader |
| 3.6 | Release: dependency-delta checklist, benchmark acceptance, `DEPRECATIONS.md` complete, v<8-freeze communicated → **cellpy 2.0** | release |

### Stage 4 — the complete system (2.0.x / 2.1)

Utils waves 3–4 (ica/ocv_rlx on curves; live/incremental rebuilt on core's engine or
deliberately dropped) · F6 feature menu (test-level summaries, `exclude_step_types`,
native-only columns in reports) · corrected IR semantics as default · volumetric
mode · docs/CLI inventory (G11) · SPEED-30 headers behind the Schema indirection ·
**2.1: shim removals per `DEPRECATIONS.md`** (prms, headers_*, property facade,
legacy curve-frame names, easyplot).

### 6.1 Safe-to-use API for util developers (start improving plotutils/batch now)

The usage report §5 showed the utils' real surface is small; the utils plan
(principle 1) promotes exactly that surface to the documented **utils contract**.
If you start improving a util *today*, code against the first column and your work
survives the flip with at most a mechanical rename pass.

**Safe — the utils contract (kept working through 2.x, shim-protected at the flip):**

| Surface | Notes for new code |
|---|---|
| `c.data` → `data.summary` / `data.steps` / `data.raw` | The only sanctioned door to the frames. Read via header *objects*, never string literals (`"voltage"`, `"cycle_index"` are what the flip renames). |
| `c.empty` | Guard before processing, as today. |
| `c.cell_name` | Titles/labels. |
| `c.get_cap` / `get_ccap` / `get_dcap` / `get_ocv` | Survive as thin wrappers over `cellpycore.curves`; signatures stable. The *returned frame's* column names get the spec'd `CurveCols` at the flip (legacy names via shim for one release) — don't hardcode `"capacity"`/`"voltage"` on results either. |
| `c.get_cycle_numbers` | Stable. |
| `c.make_summary()` / `c.make_step_table()` | Already delegate to core; stable. |
| `c.load` / `c.from_raw` / `c.save` / `c.to_csv` | Stable entry points (internals move; signatures keep). |
| `c.set_mass` / `c.set_nom_cap` / `c.mass = …` | Kept, routed through the meta facade during deprecation. |
| `c.drop_from(...)` | Returns a new object — keep the `c = c.drop_from(...)` idiom. |
| `c.cellpy_units`, `c.data.raw_units` | Fine for *reading unit values*. Use `c.data.raw_units` (not the `c.raw_units` forwarder) — one access path. Do **not** hand-compose axis-label f-strings from them; `units_label(prop, mode=…)` replaces that (wave 2). |

**Use with care (works now, changes at the flip — write it the new way when you touch it):**

- `c.headers_summary` / `headers_step_table` / `headers_normal` — become the
  attribute-level shim; after the flip the public header API is `c.schema`. Attribute
  access (`hdr.charge_capacity`) migrates mechanically; string literals don't.
- `c.data.meta_common` / meta forwarders (`c.data.nom_cap`, `.loading`,
  `.cell_name`) — become `c.data.meta.<field>` and `c.data.tests[tid].<field>`
  (facade keeps ~8 names warm through 2.0).
- pandas index idioms (`summary.loc[cycle, …]`, `raw` indexed by `data_point`,
  journal `pages.loc[cell_id]`) — the keys-in-columns rule is law and lint-enforced;
  use column filters/joins even while frames are still pandas (Phase A removes the
  indexes ahead of the flip).

**Avoid (dies without a long deprecation, or already a known trap):**

- Writing `c.ensure_step_table = True` or `c.overwrite_able = False` — these become
  load/save kwargs (utils plan principle 1).
- Mutating `prms.*` globals from utils — pass explicit kwargs; use
  `cellpy.config.override()` once it lands.
- Anything `_`-prefixed — private API breaks without deprecation (conventions §3).
- Anything in the usage report's §3 "not used by any util" lists — those members have
  **no parity guarantee** through the migration; don't add new dependencies on them
  without flagging it in the utils plan.
- Storing a `CellpyCell` in an attribute named `data` (the `ocv_rlx`
  `self.data.data.steps` trap — being renamed in wave 3).

### 6.2 Safe-to-use surface for loader developers

If you write or port a loader now, build against the loader plan's target and your
work is flip-proof:

**Safe — build on these:**

| Surface | Notes |
|---|---|
| Vendor `parse()` kept separate from normalization | The two-stage rule (§5.1.2). Your parsing is the durable part; never mix renames/casts into it. |
| Declarations in the configuration module | `column_map` (vendor→**native** names), `raw_units` (a `CellpyUnits`), `tz` rule, `reset_granularity` per cumulative column, `aux_map`, optional `post_hooks`. Validated at registration. |
| `cellpycore` toolbox | `RawCols` + `RawCols.dtype_map()` (dtype truth), `validate_raw_frame`, `Data.from_raw_frame`, `cellpycore.timestamps` (ns ↔ datetime), `StepType`/`StepMode`/`CycleType` enums, `CellpyUnits` + `validate_units`, the harmonized-raw spec (`docs/data_format_specifications/harmonized_raw.md`). |
| Draft metadata | Return a draft `TestMeta` (+ partial `CellMeta` only when the file carries masses etc.). Fill only file-known fields; leave every provenance field empty (the test-kit rejects filled ones). |
| Fixture + tests | Committed vendor sample + expected harmonized parquet (script-regenerated, F8); run `check_loader` incl. the reset-granularity property test — it is the test that catches the silent capacity-corruption bug. |

**Avoid:**

- Promoting `data_point` to a pandas index — `datapoint_num` is a column, full stop.
- Poking metadata onto `Data` attributes (`data.start_datetime = …`) — drafts only.
- Ad-hoc `raw_units` dicts or float scale factors — validated `CellpyUnits` only.
- Emitting legacy `HeadersNormal` names as your target — declare vendor→native; the
  legacy dialect exists only inside `cellpy_file/legacy_read` and the shims.
- Milliseconds or naive datetimes without a declared `tz` rule — `epoch_time_utc` is
  int64 ns UTC; the naive-datetime rule is a shared decision (native-headers D3),
  recorded on `TestMeta.time_zone`.
- Reading `prms.Instruments.*` (or any global config) inside a loader — the typed
  per-instrument model arrives as the `instrument_config` argument.
- Encoding ssh/OtherPath/remote semantics — you receive a local `Path`; the framework
  owns remote resolution (§5.1.8).

---

## 7. Iteration log

- **2026-07-09 (v1)** — initial version: total-solution synthesis, MVP vs. complete
  illustrations, design-pattern commitments, formal loader contract (§5).
- **2026-07-09 (v2)** — reconciled with the gap analysis and the six new plans:
  added the plan-set index (§0); **redefined the MVP** to match the release plan's
  2.0 support matrix (the flip — native headers + polars + v9 — is in 2.0; the old
  "keep v8/legacy dialect" MVP is now the Stage-1 preparation state); updated both
  illustrations (curves module, v9/convert, translate-at-I/O, conventions);
  extended the pattern table (declarative loader configs, two-stage pipeline,
  translate-once-at-I/O, single exception hierarchy, one deprecation cadence);
  reconciled §5 with the loader plan's `parse()`+`harmonize()` design, added the
  reset-granularity check to the test-kit, resolved the OtherPath open question
  (config plan §5b); added the timeline chapter (§6: Stages 0–4, flip gate,
  cross-repo merge order) and the safe-to-use API sub-chapters for util developers
  (§6.1) and loader developers (§6.2).
