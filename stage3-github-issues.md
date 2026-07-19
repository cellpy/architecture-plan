# Stage 3 — GitHub issues (created 2026-07-19)

Stage 3 = **2.0 assembly**: everything between the completed flip (Stage 2, shipped as
`v2.0.0a5`) and the first proper `cellpy 2.0.0` release. Stage 1 built next to the old
thing; Stage 2 swapped it; Stage 3 finalizes the contracts a major release freezes and
ships.

All issues carry the label **`cellpy2-stage3`** and the **`v2.0.0`** milestone.
Tracking issue: **[jepegit/cellpy#575](https://github.com/jepegit/cellpy/issues/575)**.
Earlier records: [stage0-github-issues.md](stage0-github-issues.md) (tracking
[#439](https://github.com/jepegit/cellpy/issues/439)),
[stage1-github-issues.md](stage1-github-issues.md) (tracking
[#459](https://github.com/jepegit/cellpy/issues/459)).

## Scope decision (2026-07-19) — what is in 2.0 and what waits for 2.1

**The test:** 2.0 finalizes what **cannot be shimmed** — frame schemas, the header API,
the loader contract, the on-disk format, metadata layout, packaging. API *sprawl* that
can ride on `warn_once` shims is cleaned in 2.1 on the conventions cadence.

The architecture plan §6 row 3.2 scheduled all four utils redesign plans into Stage 3.
That is the one place this issue set deliberately narrows the plan: **two of the four
are in 2.0.**

| Redesign plan | Verdict | Reason |
|---|---|---|
| [ica](cellpy2-ica-redesign-plan.md) | **in 2.0** | The output frame is a data contract users read columns from; changing it in a minor is a break. Smallest of the four, best characterization net (19 tests). |
| [plotting](cellpy2-plotting-redesign-plan.md) | **in 2.0** | 2.0 already breaks this surface — `easyplot` removed (#544, decision #438-5) and `batch_plotters` never ported. Ship one plotting home, or break users twice. |
| [collectors](cellpy2-collectors-redesign-plan.md) | **2.1** | Depends on the plotting redesign landing first; its collection half survives behind a facade. |
| [batch](cellpy2-batch-redesign-plan.md) | **2.1** | ~11 000 lines, the largest. User-facing surface is small and beloved — a facade holds it stable across the redesign. |

Also deferred to 2.1: utils waves 3–4 (`ocv_rlx`, live/incremental), the F6 feature
menu, SPEED-30 headers, and the shim removals registered in `DEPRECATIONS.md`.
[#303](https://github.com/jepegit/cellpy/issues/303) (cycle-statistics features) was
moved off the 2.0 milestone — additive, no contract implication.

## Loaders — [loader plan](cellpy2-loader-port-and-extraction-plan.md), [architecture plan §5](cellpy2-architecture-plan.md)

| Issue | Content | Plan |
|---|---|---|
| [#210](https://github.com/jepegit/cellpy/issues/210) | 3.0: `InstrumentLoader` Protocol + `LoaderResult` + entry-point registry + `check_loader` test-kit (existing issue, rewritten) | §5.1–5.5 |
| [#559](https://github.com/jepegit/cellpy/issues/559) | 3.2: `harmonize()` framework + declaration schema + pilot `maccor_txt` | loader §2.1–2.2 |
| [#560](https://github.com/jepegit/cellpy/issues/560) | 3.3: port tier-1/2 loaders; delete `LegacyLoaderAdapter` | loader tiers |
| [#561](https://github.com/jepegit/cellpy/issues/561) | 3.4: tier-3 decisions (biologics, batmo, ext_nda, local_instrument) | loader tier 3 |

## Public API and schema

| Issue | Content | Plan |
|---|---|---|
| [#558](https://github.com/jepegit/cellpy/issues/558) | 3.1: public `c.schema`; `headers_*` demoted to the registered shim | [native-headers](cellpy2-native-headers-migration-plan.md) Phase 4 |
| [#564](https://github.com/jepegit/cellpy/issues/564) | 3.7: `units_label()` / `with_cellpy_unit()` | [units](unit-handling-cellpy2-plan.md) Phase 4 |

## Metadata — [metadata plan](cellpy2-metadata-handling-plan.md)

| Issue | Content | Plan step |
|---|---|---|
| [#562](https://github.com/jepegit/cellpy/issues/562) | 3.5: loader drafts + `MetaResolver` with provenance | Steps 3–4 |
| [#563](https://github.com/jepegit/cellpy/issues/563) | 3.6: v9 meta persistence, merge, journal as a resolver source | Steps 5–6 |

## Configuration — [config plan](cellpy2-configuration-and-parameters-plan.md)

| Issue | Content | Plan step |
|---|---|---|
| [#565](https://github.com/jepegit/cellpy/issues/565) | 3.8: secrets env-only/`SecretStr` + generated config reference | Steps 6–7 |

## Redesigns

| Issue | Content | Plan |
|---|---|---|
| [#566](https://github.com/jepegit/cellpy/issues/566) | 3.9: `dqdv()`/`dvdq()` on one pure core, `IcaOptions`, specced output frame | [ica](cellpy2-ica-redesign-plan.md) |
| [#567](https://github.com/jepegit/cellpy/issues/567) | 3.10: plotutils as the single plotting home; `batch_plotters` retired | [plotting](cellpy2-plotting-redesign-plan.md) |

## CLI, packaging, docs, release

| Issue | Content | Plan |
|---|---|---|
| [#568](https://github.com/jepegit/cellpy/issues/568) | 3.11: library-first `cli_api` extraction (behavior-preserving) | [cli](cellpy2-cli-redesign-plan.md) Phase 0–1 |
| [#569](https://github.com/jepegit/cellpy/issues/569) | 3.12: Click → Typer cutover; `cellpy convert --to v9` | cli |
| [#570](https://github.com/jepegit/cellpy/issues/570) | 3.13: dependency budget — out `python-box`/`ruamel`/`python-dotenv`, pytables → `legacy-files` extra | [release](cellpy2-release-and-branching-plan.md) §5 |
| [#571](https://github.com/jepegit/cellpy/issues/571) | 3.14: Sphinx → Zensical, researcher-first IA | [docs](cellpy2-documentation-plan.md) |
| [#572](https://github.com/jepegit/cellpy/issues/572) | 3.15: migration guide, release notes, complete `DEPRECATIONS.md` | docs + release §1 |
| [#573](https://github.com/jepegit/cellpy/issues/573) | 3.16: lock the file-format compatibility matrix (v8 read/write, v9, convert) | release §1 |
| [#574](https://github.com/jepegit/cellpy/issues/574) | 3.17: 2.0.0 release checklist — benchmark acceptance, gates, tag | release §4/§6 |

## Sequencing constraints

1. **#210 → #559 → #560/#561.** The Protocol and registry decide where declarations
   live. The multi-test-source open question ([architecture plan §5.6](cellpy2-architecture-plan.md))
   must be answered **before the Protocol is frozen** — it cannot change after 2.0.
2. **#564 before #567** — the plotting redesign consumes `units_label()`. Small and
   additive; land it early (the utils plan already flagged this as the wave-2 blocker).
3. **#562 before #563** — persistence needs the resolved models.
4. **#568 before #569** — extract, then re-spell.
5. **#560 before the pytables move in #570** — don't reshuffle extras while the loader
   set is still moving.
6. **#573 before #574** — the release checklist asserts the matrix the compat issue builds.
7. **#572 last of the docs** — the migration guide describes what actually shipped,
   including the [§7 behavior-delta](cellpy2-architecture-plan.md) verdicts (Δ2, the CE
   inversion, is the headline item).
8. Independent, startable immediately: **#558, #564, #565, #568, #566**.

## yolo labelling

`yolo` (issue-flow hands-off mode) applied to **#558, #564, #568** — additive or
alias-guarded, crisp acceptance criteria, covered by existing oracles. Everything else
is a contract change, a decision, or a large move.
