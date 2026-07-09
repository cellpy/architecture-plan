# Stage 0 — GitHub issues (created 2026-07-09)

Stage 0 = the foundations layer: every plan's Step-0/Phase-0 characterization work,
golden fixtures, performance baselines, oracle harnesses, and shared conventions —
behavior-preserving by definition, and the precondition for starting any refactor
stage with confidence.

All issues carry the label **`cellpy2-stage0`**. Tracking issue:
**[jepegit/cellpy#439](https://github.com/jepegit/cellpy/issues/439)**.

| # | Issue | Implements | Plan document |
|---|---|---|---|
| [#428](https://github.com/jepegit/cellpy/issues/428) | Golden-fixture convention + regeneration tooling | gap F8 | [gap analysis](cellpy2-plans-gap-analysis.md), core `test-data-and-fixtures.md` |
| [#429](https://github.com/jepegit/cellpy/issues/429) | Characterization: cellpy-file round-trip + legacy matrix (incl. the limits-prefix trap) | file plan Step 0 | [file-loading plan](cellpy-file-loading-refactor-plan.md) |
| [#430](https://github.com/jepegit/cellpy/issues/430) | Characterization: configuration system + inventory parity contract | config plan Step 0 | [config plan](cellpy2-configuration-and-parameters-plan.md) |
| [#431](https://github.com/jepegit/cellpy/issues/431) | Unit groundwork: registry-interop xfail, converter parity, pint-optional guard | unit plan §6, STEP-12 | [unit plan](unit-handling-cellpy2-plan.md) |
| [#432](https://github.com/jepegit/cellpy/issues/432) | Per-loader golden snapshots of current outputs | loader plan Step 0 | [loader/extraction plan](cellpy2-loader-port-and-extraction-plan.md) |
| [#433](https://github.com/jepegit/cellpy/issues/433) | Curve-extraction golden snapshots (`get_cap` family) | extraction plan §5 | [loader/extraction plan](cellpy2-loader-port-and-extraction-plan.md), [headers report §5](hardcoded-column-headers-report.md) |
| [#434](https://github.com/jepegit/cellpy/issues/434) | Value-parity comparator through the header mapping | headers plan Phase-3 oracle | [native-headers plan](cellpy2-native-headers-migration-plan.md) |
| [#435](https://github.com/jepegit/cellpy/issues/435) | Extend consumer scans to `filters/`, `exporters/`, `internals/` | utils wave 0 / gap G5 | [utils plan](cellpy2-utils-migration-plan.md) |
| [#436](https://github.com/jepegit/cellpy/issues/436) | Benchmark harness + v1.x baselines | release plan §4 / gap G8 | [release plan](cellpy2-release-and-branching-plan.md) |
| [#437](https://github.com/jepegit/cellpy/issues/437) | Conventions bootstrap: deprecation helper, exception tree, DEPRECATIONS registry | conventions plan §5 | [conventions plan](cellpy2-conventions-plan.md) |
| [#438](https://github.com/jepegit/cellpy/issues/438) | Decision register (timezone, curve home, v9 container, IR semantics, easyplot, maintenance window) | gates Stage 1+ | multiple (listed in issue) |
| [core#114](https://github.com/cellpy/cellpy-core/issues/114) | Doc-sync pass over the cellpy-core guiding documents | gap F7 | [gap analysis](cellpy2-plans-gap-analysis.md) |

## Suggested order

`#428` first (fixture convention — everything depends on it), `core#114` + `#438` in
parallel (cheap, unblock decisions), then `#429–#433` + `#434`/`#436` in any order,
`#435`/`#437` whenever.

## Exit criteria (from #439)

- All characterization suites green on master; fast subsets marked `essential`.
- Goldens regenerate deterministically via script; no hand-edited fixtures.
- Benchmark baselines committed **before** any refactor PR merges.
- Value-parity comparator passes trivially against the current bridge.
- The six #438 decisions recorded in their plan documents.

## Status + yolo labels (updated 2026-07-09)

`#428`–`#431` **closed** (done). Remaining open issues judged fit for issue-flow's
hands-off **`yolo`** mode (well-specified, mechanical or pattern-following, low blast
radius, oracle-guarded, no open maintainer decisions) and labeled accordingly:
`#432` (already labeled by another agent), `#433`, `#434`, `#435`, `#436`, `#437`,
`core#114`. **Not** yolo: `#438` (maintainer decisions) and the `#439` umbrella.
