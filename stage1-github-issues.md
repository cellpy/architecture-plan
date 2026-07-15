# Stage 1 — GitHub issues (created 2026-07-09)

Stage 1 = **behavior-preserving construction**: every refactor and parallel build that
lands on v1.x master suite-green — cellpy-file I/O extraction, the parallel config
stack, unit consolidation, dormant header translation, literal cleanup, de-indexing —
plus the additive cellpy-core modules they consume. Nothing flips user-visible
behavior; everything is guarded by the Stage-0 oracles.

All issues carry the label **`cellpy2-stage1`**. Tracking issue:
**[jepegit/cellpy#459](https://github.com/jepegit/cellpy/issues/459)**.
Stage 0 record: [stage0-github-issues.md](stage0-github-issues.md) (tracking
[#439](https://github.com/jepegit/cellpy/issues/439)).

## cellpy-file I/O — [file-loading plan](cellpy-file-loading-refactor-plan.md)

| Issue | Content | Plan step |
|---|---|---|
| [#446](https://github.com/jepegit/cellpy/issues/446) | Constants purge + `cellpy_file/format.py` (shared with config plan; carries the limits-prefix trap warning) | file Step 1 = config Step 1 |
| [#447](https://github.com/jepegit/cellpy/issues/447) | Stateless helpers out + `LoadSelector`/`LoadResult` de-stating | Steps 2–3 |
| [#448](https://github.com/jepegit/cellpy/issues/448) | Read + write paths move; version-dispatch registry | Steps 4–5 |
| [#449](https://github.com/jepegit/cellpy/issues/449) | Out-of-band redirects, `CorruptCellpyFile`, `cellpy convert` CLI | Steps 6–7 |

## Units — [unit plan](unit-handling-cellpy2-plan.md)

| Issue | Content | Plan phase |
|---|---|---|
| [#450](https://github.com/jepegit/cellpy/issues/450) | One spec, one registry, `core`-alias rename (flips the #431 xfail to pass) | Phase 1 |
| [core#115](https://github.com/cellpy/cellpy-core/issues/115) | `convert_value`, `calculate_scaler`, `validate_units` in `cellpycore.units` | Phase 2–3 core half |
| [#451](https://github.com/jepegit/cellpy/issues/451) | Delegate the duplicated converters (deletes cellreader.py:5169–5237 body) | Phase 2 |

## Configuration — [config plan](cellpy2-configuration-and-parameters-plan.md)

| Issue | Content | Plan step |
|---|---|---|
| [#452](https://github.com/jepegit/cellpy/issues/452) | pydantic-settings stack in parallel (incl. the new `units:` section) | Step 2 |
| [#453](https://github.com/jepegit/cellpy/issues/453) | prms shim swap, 383-site migration, kill import-time init | Steps 3–4 |
| [#454](https://github.com/jepegit/cellpy/issues/454) | `cellpy setup` rewrite + `setup migrate` UX | Step 5 |

## Headers — [native-headers plan](cellpy2-native-headers-migration-plan.md)

| Issue | Content | Plan phase |
|---|---|---|
| [#455](https://github.com/jepegit/cellpy/issues/455) | Hard-coded header-literal cleanup, priorities 1–3 + dead easyplot block | Phase 0.2, [headers report §8](hardcoded-column-headers-report.md) |
| [core#116](https://github.com/cellpy/cellpy-core/issues/116) | Mapping extensions: postfix expansion + `LEGACY_ATTR_TO_SCHEMA` | Phase 1 core half |
| [#458](https://github.com/jepegit/cellpy/issues/458) | Dormant `translate.py` (`to_native`/`to_legacy`) + bridge output-shaping extraction | Phase 1 |

## Polars, conventions, core modules

| Issue | Content | Plan |
|---|---|---|
| [#457](https://github.com/jepegit/cellpy/issues/457) | Phase A de-indexing (raw/summary/journal) + warn-only index lint | [polars plan](cellpy2-polars-port-execution-plan.md) Phase A/D |
| [#456](https://github.com/jepegit/cellpy/issues/456) | Conventions machinery implementation (deprecation helper, exception tree) | [conventions plan](cellpy2-conventions-plan.md) |
| [core#117](https://github.com/cellpy/cellpy-core/issues/117) | Legacy⇄core metadata field mapping + G9/`volume` decisions | [metadata plan](cellpy2-metadata-handling-plan.md) Step 1 |
| [core#118](https://github.com/cellpy/cellpy-core/issues/118) | `cellpycore.curves` — spec'd extraction layer (#438 decision 2 recorded 2026-07-10) | [loader/extraction plan](cellpy2-loader-port-and-extraction-plan.md) §2.3 |

## Sequencing constraints

1. **#450 before #447/#448** (alias rename must not interleave with code moves).
2. **Core-first merge order**: core#115 → release+re-pin → #451; core#116 →
   release+re-pin → #458 ([release plan §3](cellpy2-release-and-branching-plan.md)).
3. **#456 before #449/#453** (they import the conventions machinery).
4. **#436 baselines before #457**.
5. #446 is one shared step of two plans; #455 and #452 can start immediately.

## Exit criteria (from #459)

Stage-0 suites still green (deliberate, documented exceptions only) · `cellpy_file/`
owns all cellpy-file I/O · one pint registry, no duplicated converters · config has no
import-time I/O and a green parity contract · `translate.py` v8→native→v8 round-trip
green and the #434 comparator green against the bridge · de-indexing done with
benchmarks in band · core releases tagged and re-pinned.

**Stage 2 preview** (not yet issued): the flip — native headers + polars in one
flag-day (headers plan Phase 3 + polars Phase C), v9 format + convert `--to v9`
(headers Phase 2), accessor shim + utils wave 1.

## yolo labels (2026-07-09)

Issues judged fit for issue-flow's hands-off **`yolo`** mode and labeled: `#446`
(constants purge + format spec — mechanical, trap documented), `#450` (registry
unification — mechanical, xfail-guarded), `#451` (converter delegation — parity-fixture
guarded; wait for core#115 re-pin), `#455` (literal cleanup — file:line tables),
`core#115`, `core#116`, `core#117` (small additive core modules following proven
patterns; #117 implements the documented G9/volume recommendations).
**Not** yolo: `#447`/`#448` (the plan's own "riskiest step" + large moves), `#449`
(multi-subsystem + perf), `#452`–`#454` (design/flag-day/UX), `#457` (cross-cutting
contract change), `#458` (blocked on #438 decisions), `core#118` (major port, gated),
`#456` (duplicate of #437 — see cross-linking comments), `#459` umbrella.

## Progress (updated 2026-07-15) — **Stage-1 issue set complete**

All remaining work items closed 2026-07-15, each via its own suite-green PR:

- **cellpy-core** (released as **cellpycore 0.2.0** on PyPI, re-pinned in
  cellpy): `core#115` units helpers (PR core#123) · `core#116` mapping
  extensions (PR core#124) · `core#117` meta_mapping + G9/volume decisions
  implemented (PR core#125) · `core#118` `cellpycore.curves` with rtol-1e-9
  parity vs the #433 snapshots (PR core#126).
- **cellpy:** `#451` converter delegation + 0.2.0 re-pin (PR #497; also fixed
  three M2 leftovers incl. the `log.setup_logging` config shadowing) ·
  `#453` closed after M1–M3 (PRs #494–#496) · `#454` setup TOML twin +
  `setup migrate` + `info --config` (PR #498) · `#457` Phase-A de-indexing +
  warn-only index lint, benchmark gate green (PR #499) · `#458` dormant
  `translate.py`, v8→native→v8 exact, #434 comparator green (PR #500).
- **#459 (umbrella) stays open** for the one remaining exit criterion: the
  behavior-delta sign-off (architecture plan §7) — maintainer decisions that
  gate the final v1.x release (architecture plan §6.1).

## Progress (updated 2026-07-14)

- **Closed:** `#446` (1.1 format spec — PR #475; golden/benchmark follow-up #476/#477),
  `#447` (1.2 stateless helpers + explicit load limits — PR #482),
  `#448` (1.3 read/write paths into `cellpy_file/` — PR #483),
  `#449` (1.4 out-of-band redirects, typed errors, `cellpy convert` CLI — PR #484),
  `#450` (1.5 one pint registry — PR #481),
  `#452` (1.7 parallel pydantic-settings stack — PR #493),
  `#455` (1.10 header-literal cleanup — PR #485),
  `#456` (1.11 conventions machinery — with Stage-0 #437),
  `#479` (1.18 easyplot deprecation — PR #480).
  A full-suite green fix landed as PR #492 after the 1.2–1.4 moves.
- **In flight:** `#453` (1.8) — M1 (prms shim + legacy YAML load path) merged as
  PR #494; M2 call-site migration in progress on branch `453-prms-m2-migrate`;
  M3 (kill import-time init) remains.
- **Open:** `#451` (waits core#115 tag + re-pin), `#454`, `#457`, `#458` (waits
  core#116), and **all four core issues** `core#115`–`core#118` — the core-first
  merge order is now the critical path; core#115/#116 should be tackled next.
- **Track E note:** the loader `harmonize()` port was deliberately *not* issued in
  Stage 1; it moved to Stage 3 (architecture plan §6, timeline correction
  2026-07-14). Only the Stage-0 goldens (#432) exist for the loaders.
- **New exit item (proposed 2026-07-14):** behavior-delta sign-off — the
  [1.0.3 vs 1.0.4a3 observations](cellpy-v103-vs-v104a3-observations.md) (CE
  inversion, coulombic sign flip, dropped columns, `discharge_c_rate` engine
  mismatch) must each be confirmed intended or fixed **before the final legacy
  release** ships; register and verdict tracking in the
  [architecture plan §7](cellpy2-architecture-plan.md). Consider adding this to
  #459's exit-criteria list.
- Benchmarks note: `#457`'s gate compares against **GHA ubuntu-latest** baselines
  (the #436 implementation, tiered after #476); local runs are timing-only, no gate.
