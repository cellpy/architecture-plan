# Gap analysis: what the cellpy 2 plans are missing

**Date:** 2026-07-09
**Method:** Cross-read of (a) the eight documents in this folder, (b) the cellpy
package surface (`cellpy/cellpy/*` — including packages no report scanned), and
(c) the cellpy-core guiding documents (`.issueflows/04-designs-and-guides/*`,
`ROADMAP.md`, the STEP-01…12 integration roadmap and its STEP-13+ issue list).
Two questions: **what has no plan at all**, and **what is already decided/documented
elsewhere but not reflected in our plans**.

Documents covered here:
usage report, hardcoded-headers report, header-swap description + excalidraw,
polars-index report, unit plan, config plan, metadata plan, file-loading plan,
native-headers migration plan.

---

## 1. Topics with no plan at all

Ordered roughly by how much they block a cellpy 2 release.

### G1 — Utils / batch / plotting migration (the user-facing 80%)

The [usage report](data-and-cellpycell-usage-in-cellpy-utils.md) inventoried *what*
utils consume, but no plan says *what happens to them* in cellpy 2. This is the
largest user surface (batch, collectors, plotutils, helpers, easyplot, ocv_rlx, ica).
Open per-module questions: port, rewrite, or deprecate (**easyplot decided:**
deprecate v1.x / remove 2.0, issue #438); where scipy-based fitting (ocv_rlx, ica) lives; how collectors change
when summaries are polars and natively richer (energies, powers, durations become
available — today's plots can't show what legacy frames don't carry).
**Action: write `cellpy2-utils-migration-plan.md`** keyed to the usage report's
per-module table, sequenced after the native-headers Phase 4.

### G2 — Instrument loaders / raw ingestion

cellpy 2 loaders must produce **harmonized raw**: native `RawCols`, `epoch_time_utc`
(int64 ns UTC), `source_*` provenance, `mask`, `test_id` — per the authoritative spec
(`docs/data_format_specifications/harmonized_raw.md`) and `RawCols.dtype_map()`
(rawcols-dtype-map.md, issue #97). No plan covers porting the ~20 loaders + their
`configurations/` modules. The unit plan touches loader `raw_units` validation and the
metadata plan changes what loaders *return* (draft `TestMeta`), but the frame-schema
change — the biggest loader change of all — is unowned. Also unowned: the fate of
`AutoLoader`/`TxtLoader` machinery and the local/custom-instrument YAML channel
(config plan §1.2 calls it "a second config channel" and stops there).
**Action: write a loader-port plan; make it the single PR series that also carries
the unit-plan Phase 3 and metadata-plan Step 3 loader changes** (three plans currently
each touch every loader once — that's three passes over 20 files instead of one).

### G3 — Polars port execution (cellpy side)

The [index report](pandas-to-polars-index-usage-report.md) is findings-only. Core's
engine migration is designed (`step-table-polars-migration.md`, issue #13), but nobody
owns the cellpy-side sequence: which frames flip first, whether pandas stays at
presentation boundaries (plotly/matplotlib take pandas happily), whether narwhals
fronts the utils, and the dual-dependency window. The native-headers plan Phase 3
assumes "rides with the polars flip" — but no document defines that flip.
**Action: promote the index report's TL;DR into an execution plan; explicitly co-plan
with native-headers Phase 3 (one flag-day, not two).**

### G4 — The extraction layer (`get_cap` and friends)

The most-used data API in the ecosystem (13+ call sites in utils alone) returns
hand-built frames with unspec'd columns (headers report §5). Core has
`extractors.py` (summary extractors) but **no cycle/capacity-curve extraction**.
Nobody has decided whether `get_cap`/`get_ccap`/`get_dcap`/`get_ocv` logic moves into
cellpy-core (natural: it is pure frame computation), stays in cellpy 2, or becomes a
core "curves" module with a spec'd schema. This decision gates G1 and the
curve-header fix the native-headers plan lists as a prerequisite.
**Action: decision + small design doc; recommend a core `curves.py` with spec'd
output schema, mirroring how `extractors.py` was introduced (issue #23 pattern).**

### G5 — `cellpy/filters/` — scanned by nobody

`cellpy/filters/{cycles,summary}.py` exists and appears in the polars report's
mask findings (§2.12), but the usage report scanned only `utils/`, the hardcoded
report didn't classify it as a consumer area, and no migration plan mentions it.
Same blind spot: **`cellpy/exporters/`** and **`cellpy/internals/`** (OtherPath —
see G7). **Action: extend the usage/hardcoded scans to these packages; fold their
migration into G1's plan.**

### G6 — Error handling, logging, and warnings policy

Core has a clean `exceptions.py`; cellpy has `cellpy/exceptions.py` plus ad-hoc bare
`Exception`s (file plan fixes exactly one: `CorruptCellpyFile`). No plan defines the
cellpy 2 exception hierarchy, the logging setup (today: `log.py` + `logging.json`,
import-time configuration — same import-time-side-effect smell the config plan kills
for prms), or a deprecation-warning policy (five plans each invent shims with
warnings; nobody defines the shared cadence: warn in v2.0, remove in v2.1?).
**Action: one short conventions doc; adopt it in every shim the plans create.**

### G7 — OtherPath / remote data access

`internals/otherpath.py` (ssh/scp-backed paths, `.env` credentials) is load-bearing
for the file plan (external-file temp copy), the config plan (secrets), and every
loader — but its cellpy 2 future (keep? replace with fsspec? core-adjacent?) is
undecided. **Action: explicit decision; cheap now, expensive after G2 hard-codes it
into 20 loaders.**

### G8 — Performance acceptance and benchmarks

"Small, fast" is cellpy-core's raison d'être, and the bridge pays 3–5 renames +
pandas↔polars conversions per call that cellpy 2 removes — but no plan defines a
benchmark suite or acceptance numbers (load → steps → summary wall-time on the golden
cells; batch of N cells). Without it, "the polars port made things faster" is
folklore. **Action: tiny benchmark harness in CI, baselined on v1.x before Phase-3
flips.**

### G9 — Packaging, versioning, and the support matrix

Scattered fragments exist (config plan drops box/ruamel/dotenv; file plan freezes
v<8; release-procedure.md covers core's PyPI flow) but no plan states the cellpy 2
release strategy: version number (2.0), branch strategy (the mine-don't-merge lesson
from branch 334 argues for trunk-based with flags, not a long-lived v2 branch),
python floor communication, the pandas+polars dual window, and the v1.x maintenance
promise (how long do v8-writes stay supported?). **Action: one-page release/branching
doc; decide long-lived-branch vs feature-flags explicitly.**

### G10 — Live / incremental processing

`utils/live.py` + `utils/processor.py` exist in cellpy; core's ROADMAP marks
**incremental summarization as DONE** in the engine. No plan connects them — the
cellpy 2 story for "poll a running test and refresh steps/summary incrementally"
would be a headline feature and the core support already exists. **Action: fold into
G1's utils plan as an explicit module, or drop the modules deliberately.**

### G11 — Docs, templates, example data

`cellpy setup` migration is planned (config plan Step 5); the rest of the CLI
(`new`, `edit`, `pull` templates), the cookiecutter template registry, hosted
example-data (`example_data.py` downloads), and the user-docs rewrite for changed
APIs are unplanned. Core solved its docs tooling (zensical-docs.md) — decide whether
cellpy 2 follows. **Action: docs/CLI inventory pass; can trail everything else.**

### G12 — Third-party extension API: loader plugins + exporter contract *(added 2026-07-09)*

Surfaced by a hypothetical-demands review ("a user wants to contribute a loader and an
exporter"). Two holes: **loader plugin discovery is a literal TODO**
(`find_all_instruments`, data_structures.py:985 — "Searching for modules through
plug-ins: Not implemented yet"), so today a third-party loader requires a PR into
cellpy itself; and **no exporter contract exists anywhere** — `to_csv`/`to_excel` are
`CellpyCell` methods, `exporters/` was the unscanned package (G5), and the BDF
prototype sits in core's `scripts/bdf/` with placement parked. The fix is cheap if
specified **before** the loader port hard-codes its configurations (Stage 3), and
expensive after. **Action: done — `cellpy.loaders`/`cellpy.exporters` entry-point
groups, registration-time validation, the plain-function exporter contract over native
`Data`, contributor kit, and BDF-as-reference are now §2.4 + Step 7 of the
[loader/extraction plan](cellpy2-loader-port-and-extraction-plan.md).**

---

## 2. Documented elsewhere, forgotten in our plans

### F1 — SPEED-30: unit+dtype-carrying, versioned headers (`SuperDuperCols`)

`column-headers-review.md` (§F) recommends a header object carrying
`value + unit + dtype` per column, prototyped in `dev/col_structure_development.py`.
**Two of our plans re-invent slices of it**: the unit plan's `df.attrs["units"]`
(Phase 4) and the native-headers plan's schema stamping. These must converge on one
mechanism — recommendation: keep both plans' short-term devices, but name SPEED-30 as
the end state and design `units_label()` so its lookup can be re-pointed at
SPEED-30 headers later. *(Noted in native-headers plan D5; unit plan not yet
updated.)*

### F2 — Step-type vocabulary already has a home

The hardcoded report (§7) deferred step-type literals ("worth its own pass") — but
core issue #24 already delivered it: `StepType`/`StepMode`/`CycleType` StrEnums,
`STEP_TYPES` single source in `cellpycore.config`. The pass is therefore *mechanical
adoption*, not design. Also inherited from #24: the open `""` vs `NOT_KNOWN`
unification and the likely `cycle_type` → `test_type` metadata migration — the
latter belongs on the **metadata plan's** open-questions list and isn't there.

### F3 — Naming decisions with downstream reach: `potential`, seconds

Issue #10 fixed `potential` (not voltage) and `test_time` in **seconds** as canonical.
Consequences not yet absorbed: the unit plan's `units_label("voltage")` vocabulary
should be `potential` post-swap (or alias both); any loader emitting milliseconds must
convert at ingestion (G2's spec). Small, but the kind of thing that silently produces
mislabeled axes.

### F4 — The IR-semantics bug is *frozen into the oracle*

`summary-extractors.md`: legacy `_ir_to_summary` is documented-wrong
(`# THIS DOES NOT WORK PROPERLY!!!!`), the buggy behaviour is preserved for parity,
and `extractors.py` ships a corrected extractor. **No plan decides when cellpy 2
switches to the corrected semantics.** This matters for the native-headers plan's
Phase-3 oracle ("value parity on mapped columns" would *fail* on `ir_charge` if the
corrected extractor is the default) — the switch must be an explicit, documented
oracle exception. Same pattern applies to any other frozen bug the goldens encode.

### Decision (2026-07-10, issue #438) — IR-semantics switch

Cellpy 2 adopts the **corrected** IR extractor at the **native-headers flip (Phase 3)**.
`ir_charge` / `ir_discharge` (and related summary IR columns) are on the value-parity
**exception list** in `tests/parity.py` until goldens are updated or the shim documents
the intentional divergence. No IR semantic change before the flip — Stage 0 oracles stay
frozen.

### F5 — Engine gaps tracked in core that G2 will trip over

- **Reset-granularity normalization** (issue #42): legacy loaders deliver per-step or
  per-test cumulative capacities inconsistently; the engine expects one convention.
  The loader-port plan (G2) must map each instrument to its reset behaviour.
- **`ref_potential` support** (issue #43): deliberately not bridged; loaders with
  reference electrodes need the native-path answer.
- **`exclude_step_types`** (issue #54): the roadmap table says "⬜ future", but the
  code exists (`summarizers.py:655/715`, threaded through `make_core_summary`) —
  see F7. cellpy 2 should expose it (it replaces the old selector-exclusion feature
  users lost in #45).

### F6 — ROADMAP features with no consumer plan

Core's ROADMAP: **test-level summaries** (`build_tests_summary`) planned;
**merging** and **incremental summarization** marked DONE. Our metadata plan uses the
merge scaffolding, but nothing plans the cellpy 2 user API for merged tests
(`c.tests` summaries?) or incremental refresh (G10). Features that exist in the
engine but never surface in the app are wasted work — put them on the utils plan's
menu explicitly, even if the decision is "not in 2.0".

### F7 — Status drift in the guiding docs themselves

Found while cross-reading (each small, together corrosive):

- Integration roadmap STEP-12 says "🟡 partly — schema lives in `legacy.py`"; in
  reality `cellpycore/units/spec.py` exists (issue #40 executed, plus #112) and the
  duplicate-converter retirement is exactly our unit plan Phase 2.
- Roadmap STEP-13+ table: #54 marked future but implemented (see F5).
- `column-headers-review.md` §"Issue #34" points at `src/cellpycore/header_mapping.py`;
  the module now lives at `src/cellpycore/legacy/mapping.py`.
- `cellpy-core-integration-into-cellpy.md` "Key findings" still says
  `make_step_table` is NOT ported (superseded by STEP-08 ✅ in the same folder).

**Action: a one-hour doc-sync pass in cellpy-core after each milestone; stale
"partly done" statuses cost every future planner a re-derivation** (this gap analysis
included).

### F8 — Test data & fixtures philosophy exists; our plans each re-invent it

`test-data-and-fixtures.md` explains core's approach (committed golden parquet, no
instrument loaders needed, `dev/regenerate_test_data.py` for intentional
regeneration). Our file-loading, header-migration, and unit plans each specify
characterization/parity tests independently. Adopt the same mechanics cellpy-side:
golden fixtures regenerated by script, never by hand — one shared
`tests/data/` convention for the cellpy 2 work.

### F9 — Release/pinning procedure

`release-procedure.md` documents tag → PyPI (`cellpycore`) → cellpy re-pin.
Our plans all assume the editable `[tool.uv.sources]` wiring; none mentions that
landing a cellpy PR that *requires* new core features means **core releases first,
cellpy re-pins second** (merge-order lesson recorded in #45: cellpy PR first only for
*removals*). Add the cross-repo merge-order rule to each plan that touches both repos
(unit Phase 2, metadata Steps 1–2, header-migration Phase 1).

### F10 — Stage-0 platform strategy (UUIDs, BattINFO/EMMO)

`column-headers-review.md` §F records the multi-scale strategy needs: stable UUIDs
for cells/electrodes/materials/protocols/test-runs and BattINFO/EMMO term mappings
per column. The metadata plan covers `TestMeta.uuid` only. Nobody carries the
cell-identity scheme (`CellMeta` has no uuid) or the ontology mapping. Fine to defer
— but it should be deferred *on a list*, not forgotten: park both on the metadata
plan's open questions.

---

## 3. Ownership and sequencing summary

| # | Item | Owner document | Status (2026-07-09) |
|---|---|---|---|
| G2+G4 | Loader port + extraction layer decision | [cellpy2-loader-port-and-extraction-plan.md](cellpy2-loader-port-and-extraction-plan.md) | ✅ plan written |
| G1+G5+G10 | Utils/filters/exporters/live migration | [cellpy2-utils-migration-plan.md](cellpy2-utils-migration-plan.md) | ✅ plan written |
| G3 | Polars execution (cellpy side) | [cellpy2-polars-port-execution-plan.md](cellpy2-polars-port-execution-plan.md) | ✅ plan written |
| G6 | Conventions (exceptions/logging/deprecations) | [cellpy2-conventions-plan.md](cellpy2-conventions-plan.md) | ✅ plan written |
| G9+G8+F9 | Release, branching, benchmarks, merge order | [cellpy2-release-and-branching-plan.md](cellpy2-release-and-branching-plan.md) | ✅ plan written |
| G7 | OtherPath decision | config plan §5b addendum | ✅ recorded |
| G11 | Docs/CLI inventory | release plan §6 (trailing) | parked there |
| G12 | Extension API: loader entry points + exporter contract | loader/extraction plan §2.4 + Step 7 | ✅ recorded (2026-07-09) |
| F1 | SPEED-30 convergence note | unit plan §8 + native-headers plan D5 | ✅ both recorded |
| F2, F3, F5 | Adopt enums / naming / engine gaps | loader + utils plans inherit | ✅ folded in |
| F4 | IR-semantics switch decision | native-headers plan (Phase-3 oracle exception) | ✅ decided (2026-07-10, issue #438) |
| F6 | Feature-surface menu | utils plan §1.4 / wave 4 | ✅ folded in |
| F7 | Doc-sync pass | cellpy-core chore | **still open — one hour, do it** |
| F8 | Shared fixture convention | loader plan Step 0 + all test sections | ✅ adopted in new plans |
| F10 | UUID/ontology parking | metadata plan open question 6 | ✅ recorded |

The two biggest structural findings: **the loader port (G2) is the unowned
critical-path item** — three existing plans each schedule a pass over the same 20
files, and the schema change that dwarfs all three has no home; and **the
extraction layer (G4)** gates both the utils migration and the curve-header
prerequisite that the native-headers plan already depends on.
