# Plan: cellpy 2 release, branching, benchmarks (G9 + G8 + F9)

**Date:** 2026-07-09
**Closes:** gap-analysis items **G9** (packaging/versioning/support matrix),
**G8** (benchmarks), **F9** (cross-repo merge order), with **G11** (docs/CLI) as a
trailing note.

---

## 1. Versioning and support matrix

| | cellpy 1.x | cellpy 2.x |
|---|---|---|
| Python | ≥3.13 (already raised, STEP-01) | ≥3.13 |
| Frames | pandas, legacy headers | polars, native headers |
| cellpy file | reads v3–v8, writes v8 | **reads v8 + v9, writes v9** (+ `save(format="v8")` compat); v<8 via `cellpy convert` on 1.x |
| cellpy-core | pinned release (bridge/`OldCellpyCellCore`) | pinned release (native `CellpyCellCore`) |
| Maintenance | bugfix-only from 2.0 release; **12 months** from 2.0 release date (issue #438) | mainline |

### Decision (2026-07-10, issue #438) — v1.x maintenance window

After cellpy 2.0 ships, the `v1.x` fork receives **bugfix-only** support for **12
months** from the 2.0 release date (not last-two-minors). Communicate in 2.0 release
notes and 1.x deprecation warnings.

Communicate the v<8 freeze and the convert path in the 2.0 release notes *and* in the
1.x deprecation warnings (users hear it before 2.0 exists).

**Release-notes gate for the final v1.x release (added 2026-07-14):** the observed
1.0.3 → 1.0.4a3 behavior deltas — CE inversion, coulombic-difference sign flip,
dropped `shifted_*` specific and `reference_voltage_*` columns, the fixed step
misclassification — must each be confirmed intended and release-noted (or fixed)
before the final legacy release ships. Register with verdicts:
[architecture plan §7](cellpy2-architecture-plan.md); source data:
[cellpy-v103-vs-v104a3-observations.md](cellpy-v103-vs-v104a3-observations.md).
The final-legacy ship gate itself is architecture plan §6.1.

## 2. Branch strategy — trunk-based, short-lived flip branches

**Recommendation: no long-lived `v2` branch.** The branch-334 lesson
(`cellpy-core-integration-into-cellpy.md`: 136 ahead / 24 behind, overlapping
rewrites, "mine, don't merge") is exactly the failure mode a v2 branch reproduces at
larger scale. Instead:

- All preparation that is behavior-preserving lands on master behind the existing
  seams (file-loading refactor, de-indexing Phase A, translate.py dormant, config
  stack parallel build — the plans are already shaped this way).
- The two genuine flag-days (native-headers Phase 3 + polars Phase C — deliberately
  the same day) live on a **short-lived** feature branch, merged when green against
  the value-parity oracle.
- 1.x maintenance forks as `v1.x` **at** the flip, not before.
- Revisit trigger: if the flip branch lives >4 weeks, stop and re-slice; do not let
  it become branch 334.

## 3. Cross-repo merge order (F9 — the rule every dual-repo plan inherits)

- **Additions** (cellpy needs new core API): core PR → core release
  (`release-procedure.md`: bump, tag, PyPI) → cellpy re-pins → cellpy PR.
  Editable `[tool.uv.sources]` wiring is for dev loops only; CI and releases pin.
- **Removals** (core drops what only cellpy used): cellpy migrates off first, then
  core deletes — the documented #45 order.
- Applies to: unit plan Phase 2 (`convert_value`/`calculate_scaler` are additions),
  metadata plan Steps 1–2, header-migration Phase 1 (mapping extensions),
  loader/extraction plan Step 2 (`cellpycore.curves`).

## 4. Benchmarks (G8)

- **Harness:** `pytest-benchmark` (stays inside the existing test tooling; asv is
  overkill for two repos this size), `benchmarks/` collected but excluded from the
  default run, executed by a dedicated CI job on a fixed runner class.
- **Metrics** (on the committed golden cells): single-cell `load → make_step_table →
  make_summary` wall time; 20-cell batch summary collection; v8 load and v9 load;
  `get_cap` over all cycles. Peak RSS recorded where cheap.
- **Baselines:** capture on v1.x **before** polars Phase A starts (the gap analysis
  point: without a baseline, "faster" is folklore). Store results as committed JSON;
  the CI job compares ±20% and fails on regression outside the band.
  *(Implemented 2026-07-10, issue #436: baselines are captured on GitHub Actions
  ubuntu-latest — never on local machines — so the ruler matches the compare
  runner; a tiered gate replaced the flat band after #476; peak RSS is
  informational-only; the v9-load benchmark is deferred until the format lands.)*
- **Acceptance for 2.0:** no metric slower than 1.x; the flip (bridge removal +
  parquet) is expected to show a measurable win on the load and summary paths — if it
  doesn't, investigate before release.

## 5. Dependency budget bookkeeping

Target deltas for 2.0 (each owned by an existing plan, gathered here so the release
notes and `pyproject.toml` review have one checklist): **out** — python-box, ruamel
(post-migration window), python-dotenv, pytables/HDF5 as a *required* dep (moves to a
`legacy-files` extra with the v8 path); **in** — pydantic-settings, platformdirs,
polars (via core), pyarrow/parquet. pandas stays (plot/Excel boundaries) — removing
it is explicitly *not* a 2.0 goal.

## 6. Trailing: docs and CLI (G11)

Not release-blocking; owned by dedicated plans (written 2026-07-17):

- [cellpy2-documentation-plan.md](cellpy2-documentation-plan.md) — Sphinx→Zensical
  (follow cellpy-core), researcher-first information architecture, docstring-driven
  API reference cleanup, example notebook re-render policy.
- [cellpy2-cli-redesign-plan.md](cellpy2-cli-redesign-plan.md) — Click→Typer;
  extract library-first `cli_api` so `setup` / `run` / `pull` / `new` / `convert`
  are callable from scripts; config Step 5 still owns the *config* side of `setup`.

Generated config + deprecation tables still come from the config / conventions
plans; v9 example-data regeneration stays in the utils plan. Sequence: CLI Phase 0–1
(behavior-preserving extract) can start before the flip; docs content rewrite and
Typer cutover trail Stage 3 API stabilization.
