# architecture-plan

Authoritative **cellpy 2 architecture and migration plans** for the
`cellpy-workspace` multi-repo checkout.

## What lives here

Topic plans (configuration, loaders, units, native headers, polars port, utils
migration, release/branching, conventions), the coordinating
[`cellpy2-architecture-plan.md`](cellpy2-architecture-plan.md), gap analysis
([`cellpy2-plans-gap-analysis.md`](cellpy2-plans-gap-analysis.md)), and Stage 0 issue
notes ([`stage0-github-issues.md`](stage0-github-issues.md)).

## Sibling repositories

| Checkout | Role |
|----------|------|
| `../cellpy/` | Consumer library; issue-flow under `.issueflows/` |
| `../cellpy-core/` | Compute engine |
| `architecture-plan/` (this repo) | Plan documents |

## Note for agents

Plans **used to** live under a `code-reviews/` folder in the workspace. That is no
longer correct — read and cite paths from **`architecture-plan/`** (this repository).

Cross-reference in cellpy:
`cellpy/.issueflows/04-designs-and-guides/cellpy-workspace-repos.md`.
