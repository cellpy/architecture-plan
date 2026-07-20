# Plan: documentation redesign (Zensical, researcher-first, clean API refs)

**Date:** 2026-07-17 (decisions locked same day — see §7)
**Closes:** the docs half of gap-analysis item **G11** (release plan §6:
"decide whether cellpy 2 adopts core's zensical docs tooling"; user-docs
rewrite for changed APIs).
**Direction (maintainer request):** migrate the docs stack to **Zensical**;
clean up and update content; restructure for the real audience (**battery
researchers**, not software developers); clean up API documentation — which
mostly means cleaning up **docstrings in the codebase**.
**Foundation:** cellpy-core
`cellpy-core/.issueflows/04-designs-and-guides/zensical-docs.md` (tooling
precedent), [conventions](cellpy2-conventions-plan.md) (Google-style
docstrings — already the cellpy-core rule), [config plan](cellpy2-configuration-and-parameters-plan.md)
Step 7 (generated config docs), [CLI plan](cellpy2-cli-redesign-plan.md)
(CLI reference + script recipes), every topic plan's "docs/symbol map" notes
(batch, plotting, ICA, utils, native-headers).

---

## Implementation record (2026-07-19, cellpy#571/#589)

Sphinx → Zensical landed. `zensical.toml` with the researcher-first nav
(Getting started / Tutorials / How-to / Concepts / Reference / Development);
mkdocstrings/Griffe API pages replacing 14 hand-written RST stubs (static
analysis — the RTD build installs only zensical + mkdocstrings-python, not
cellpy); `.readthedocs.yaml` on `build.jobs`; a dedicated docs CI workflow
whose link check matches the issue *count* (a bare "issues found" grep also
matches "No issues found" — it failed its own clean build first).

Notebook rendering: a straight nbconvert produced ~50 MB of markdown because
plotly embeds a self-contained HTML+JS blob per figure;
`dev/render_example_notebooks.py` strips the interactive payloads and keeps
the static PNGs (148 KB committed). It renders the outputs the notebooks
already carry — it does **not** execute them, so "the docs build" is not
quietly standing in for "the tutorials run"; executing them is separate work.

Local build: 51 pages, no issues. Post-#571 additions ride the same rails
(api/ica.md in #590, api/plotting.md in #595).


## 1. Where documentation lives today

### 1.1 Tooling

| Piece | Today |
|---|---|
| Builder | **Sphinx** (`docs/conf.py`) + MyST + nbsphinx + autodoc/napoleon |
| Hosting | Read the Docs (`docs/environment_rtd.yml`, `docs/requirements_doc.txt`) |
| API refs | Hand-maintained `docs/source/cellpy*.rst` stubs + autodoc |
| Notebooks | nbsphinx executes/embeds `.ipynb` under `docs/examples/` |
| Config | Sphinx-centric; Windows-hostile path inserts in `conf.py`; lazy-import note already blocked autoapi |

cellpy-core already chose **Zensical** (Material-for-MkDocs successor) +
mkdocstrings + committed rendered markdown for examples. Cellpy 2 follows
that **builder** stack; example *sources* are **marimo** notebooks (not
Jupyter-as-authoring — see §3.4), with downloads in both marimo and
`.ipynb` form.

### 1.2 Content map (current)

```
docs/
  index.md                 # includes adapted_readme.md
  getting_started/         # install, config, checkup, basic_usage, migration
  fundamentals/            # data structure, file formats, …
  examples/                # notebooks + batch_utility tutorial
  contributing/            # developer guides (packaging, loaders, setup, …)
  other/                   # authors, license, history, header migration map
  source/                  # Sphinx autodoc RST stubs
```

### 1.3 What is wrong

1. **Audience skew.** Large `contributing/developers_guide/` and packaging
   how-tos sit at the same nav depth as "load a file and plot capacity."
   A battery researcher landing on the docs meets software-process pages
   before a minimal workflow.
2. **Stale / overlapping pages.** Concepts repeated across README,
   `adapted_readme.md`, getting_started, and fundamentals; v1-era names
   (`prms`, old headers, easyplot) still present; migration notes mixed
   into evergreen guides.
3. **API reference quality tracks docstring quality.** Autodoc cannot fix
   missing Args/Returns, wrong parameter names, copy-pasted Keyword Args
   blocks (ICA plan §1.1), or private helpers exported because an `.rst`
   stub listed them. Cleaning "the API docs" **is** cleaning public
   docstrings + choosing a public export set.
4. **Sphinx cost.** Heavy RTD env (project + jupyter + nbsphinx), fragile
   with lazy imports, two markup dialects (RST stubs + MyST). Core's
   Zensical RTD build needs only `zensical` (+ mkdocstrings for API).
5. **No single "day-1 path".** Install → configure → load one cell →
   summary plot → save is discoverable only if you already know cellpy.

### 1.4 What is genuinely good (and must survive)

- Example notebooks that show real workflows (loading, capacity vs voltage,
  ICA, GITT, batch) — content survives; authoring moves to marimo (§3.4).
- `getting_started/migration_v1_to_v2.md` as an explicit bridge page.
- Fundamentals material on file formats / data structure — keep, rewrite
  in researcher language, **link out** to core specs for normative detail.
- RTD hosting URL continuity (`cellpy.readthedocs.io`), including the
  existing v1.x branch version.

---

## 2. Design principles

1. **Researcher-first nav.** The default path assumes a battery scientist
   who knows cells and cyclers, not packaging or entry points. Developer
   material lives under **Development** (secondary).
2. **One day-1 story.** Home → install → configure → first analysis
   (load → steps/summary → one plot → save) in under 15 minutes of reading.
3. **Zensical on RTD; examples pre-rendered.** Same builder patterns as
   core (`zensical.toml`, mkdocstrings Google-style). RTD never executes
   notebooks — a GitHub Action renders marimo → committed markdown (+
   assets). Hard cut from Sphinx; v1.x docs stay on the RTD **v1.x branch**
   (already configured).
4. **Marimo is the example source of truth.** Author and maintain examples
   as marimo notebooks; offer **both** marimo and Jupyter (`.ipynb`)
   downloads on each example page (export from marimo).
5. **Docstrings are the API docs.** mkdocstrings renders what is in the
   code. Public modules get careful Google-style docstrings; private
   (`_`-prefixed) and legacy shim surfaces are excluded or marked
   deprecated. Enforcement for 2.0 is **review-only**; lint later (§8).
6. **Generated where truth is structured.** Config reference from pydantic
   models (config plan Step 7); deprecation table from
   `DEPRECATIONS.md` / `warn_once` registry (conventions plan); CLI help
   from Typer (CLI plan). Do not hand-maintain duplicates.
7. **Evergreen vs migration.** v1→v2 migration, header maps, and "what
   disappeared" live in a **Migration** section that can shrink after 2.1;
   they do not pollute getting-started forever.
8. **Short README, rich docs Home.** README is a thin GitHub front door
   (pitch + install + link to RTD). No `adapted_readme` include — the RTD
   Home is the real landing page.

---

## 3. Proposed information architecture

Target nav (names indicative; final titles can be friendlier):

```
Home
Getting started
  ├── Installation
  ├── Configuration (user-facing; links to generated reference)
  ├── Your first cell (load → summary → plot → save)
  └── Checkup / troubleshooting
User guide
  ├── Loading data (instruments, cellpy files, units)
  ├── Exploring a cell (steps, summary, cycles, curves)
  ├── Plotting
  ├── Batch workflows
  ├── ICA / DVA
  ├── Metadata & journals
  └── Automating cellpy (cli_api + CLI; from CLI plan)
Examples                    # rendered md; download marimo + .ipynb
Migration (v1 → v2)         # symbol maps, behavior deltas, header map
Reference
  ├── Configuration reference (generated)
  ├── CLI reference
  ├── Deprecations
  └── API                         # mkdocstrings; curated public story only
Specifications              # thin wrappers; normative detail on cellpy-core RTD
  └── (harmonized raw, file format, …) — link out, do not mirror
Development                 # contributing, loaders/plugins, release, docs build
Changelog
```

### 3.1 Audience split

| Audience | Primary sections | Avoid dumping here |
|---|---|---|
| Battery researcher | Getting started, User guide, Examples, Migration | Packaging, conda-forge recipes, internal folder maps |
| Lab power user / scripter | Automating cellpy, CLI, Batch, Reference | — |
| Contributor | Development, Specifications, API internals | Home / Getting started |

### 3.2 Tooling layout (repo)

```
zensical.toml                 # repo root (like cellpy-core)
docs/
  index.md                    # real Home (not adapted_readme)
  getting-started/
  user-guide/
  examples/
    *.py                      # marimo sources (authoring truth)
    *.md + *_files/           # committed renders (GHA-produced)
    exports/*.ipynb           # committed Jupyter exports for download
  migration/
  reference/
    config.md                 # generated or --8<-- include
    cli.md
    deprecations.md           # --8<-- DEPRECATIONS.md
    api/                      # mkdocstrings pages (allowlisted)
  specifications/             # thin pages + links to core RTD
  development/
.readthedocs.yaml             # zensical-only (hard cut; v1.x branch separate)
.github/workflows/
  docs-examples.yml           # run marimo → md + ipynb; commit or PR artifacts
```

Remove (after cutover): `docs/conf.py`, `docs/Makefile` / `make.bat`,
`docs/source/*.rst`, `docs/adapted_readme.md`, Sphinx-only requirements.
Keep `docs/development/docs-build.md`: local `zensical serve`, how to run
the example-render workflow, marimo authoring notes.

### 3.3 API surface for mkdocstrings

Curate explicitly — do **not** autodoc the entire package tree:

**In (2.0 public story):** `cellpy` top-level convenience, `CellpyCell` (or
successor facade), `cellpy.batch`, `cellpy.plotting`, `cellpy.cli_api`,
config access, key readers entry points, ICA/`dqdv`/`dvdq`.

**Out / deferred:** `parameters.legacy`, private `_` modules,
`libs/`, farm/barn internals once batch redesign lands, shim modules beyond
a single "Compatibility" page that points at Migration.

Docstring standard (**review-only for 2.0**; lint is §8 future work):

- Google style (`Args:`, `Returns:`, `Raises:`, `Example:`, `Note:`) — same
  as cellpy-core's project rule.
- First line = imperative summary; no REST/Numpy styles.
- Public functions document units and frame schemas (column names) when they
  return tables — link out to core specs rather than duplicating long tables.
- Shims' docstrings state replacement + removal version (conventions plan).

### 3.4 Example notebook policy (marimo)

Zensical does not execute notebooks at RTD build time. Authoring and
rendering:

1. **Source of truth:** marimo notebooks under `docs/examples/` (typically
   `.py` marimo files). Port existing Jupyter examples to marimo as part of
   T6; do not keep a dual-authoring workflow.
2. **GitHub Action** (e.g. `docs-examples.yml`): on change to example
   sources (and on a schedule or `workflow_dispatch`), run each notebook,
   export **markdown** (+ figure assets) and **`.ipynb`**, then commit or
   open a bot PR with the artifacts. RTD only reads the committed renders.
3. **Downloads on each example page:** both formats — marimo source and
   Jupyter `.ipynb` — so researchers on either toolchain can grab a copy.
4. **Nav** links to the rendered markdown; download buttons/links for the
   two source formats.
5. Feature waves (batch/plotting/ICA) update the relevant marimo notebooks
   in the same PR series; the Action refreshes committed renders.

---

## 4. Content workstreams (not just tooling)

Tooling migration without content work fails the maintainer request. Parallel
tracks:

| Track | Work | Depends on |
|---|---|---|
| **T0 Tooling** | Zensical skeleton, RTD, mkdocstrings hello-world | none |
| **T1 Day-1 path** | Rewrite install / config / first-cell against 2.0 APIs | config + plotting MVP |
| **T2 User guide** | One page per workflow; delete duplicates; researcher voice | utils/batch/plotting/ICA plans' public APIs |
| **T3 Docstrings** | Pass over public surface; fix Args/Returns; remove ghost params | API freeze per module as it lands in 2.0 |
| **T4 Migration** | Symbol maps from batch/plotting/utils plans; header map; behavior deltas (§7 architecture plan) | those plans' tables |
| **T5 Generated refs** | Config from models; DEPRECATIONS; CLI help | config Step 7, conventions, CLI Phase 3 |
| **T6 Examples** | Port examples to marimo (2.0 APIs); GHA renders md + ipynb | T1–T2 APIs |

Voice guidelines (short):

- Prefer "load your Arbin file" over "invoke the instrument loader plugin".
- Show units in examples (`mAh/g`, not only column keys).
- One concept per page; link deeper specs instead of inlining engine detail.
- Code cells that run as copy-paste in a fresh environment (or clearly mark
  optional extras).

---

## 5. Migration plan

Trailing relative to the Stage-2 flip, but **T0 can land early** on a branch
without waiting for every API. Content tracks ride the feature PRs.

### Phase 0 — skeleton (1–2 days)

1. Add `zensical.toml`, `docs/` tree stub, `[dependency-groups] docs` with
   `zensical`, `mkdocstrings-python`, `marimo` (and whatever the Action needs
   to export md + ipynb).
2. Port Home + Getting started stubs (even if still v1 APIs) to prove RTD.
3. **Hard cut:** point RTD `main`/2.0 builds at Zensical only. v1.x docs
   remain on the RTD **v1.x branch** (already configured — no dual Sphinx
   build on the new branch). Preview via RTD PR builds, then merge.

### Phase 1 — structure + move (2–3 days)

1. Rehome markdown into the IA in §3; leave TODO banners on pages that still
   describe v1-only APIs.
2. Demote developer guides under Development; drop obsolete packaging pages
   or mark historical.
3. Wire mkdocstrings for a **small** public set (`cellpy` / `cli_api` /
   config) so the API section exists.

### Phase 2 — docstring campaign (ongoing, ~1 week concentrated)

Per public module, as it stabilizes for 2.0:

1. List public names (explicit `__all__` or mkdocstrings allowlist).
2. Fix docstrings to Google style; delete documented-but-removed params.
3. Add one minimal `Example:` where it unblocks researchers (load, plot,
   batch) — not on every helper.
4. PR checklist line: *"mkdocstrings page reviewed in the preview"*.

Coordinate with feature work: plotting/batch/ICA PRs update their own
docstrings; this phase is the sweep for leftovers (`CellpyCell`, readers,
config).

### Phase 3 — content rewrite (1–2 weeks, can overlap Phase 2)

1. Day-1 path against 2.0; slim README to front-door + RTD link (drop
   `adapted_readme`).
2. User-guide pages; delete or redirect stale duplicates.
3. Migration section filled from plan symbol maps + architecture §7 deltas.
4. Specifications section = thin wrappers linking to cellpy-core RTD.
5. Examples: marimo port + wire `docs-examples` Action; commit first render
   set (md + ipynb exports).

### Phase 4 — generated references + CLI (2–3 days)

Config tables, DEPRECATIONS include, CLI reference, "Automating cellpy"
recipes (CLI plan Phase 3).

### Phase 5 — remove Sphinx debt

Delete conf/RST/requirements/`adapted_readme`; update CONTRIBUTING / README
badges; confirm RTD `main` only runs Zensical (v1.x branch untouched).

Total: **~3–4 weeks** calendar if overlapped with Stage 3 feature docs; pure
docs critical path ~2 weeks after APIs stabilize enough for day-1.

---

## 6. Risks

| Risk | Mitigation |
|---|---|
| Docs describe 2.0 APIs before they exist | TODO banners + Migration section; day-1 page merges when plotting+load path is real |
| mkdocstrings pulls private junk | Strict allowlists / explicit API pages; `__all__` discipline |
| Example render bitrot | GHA fails (or opens a PR) when marimo sources change and renders are stale; keep fixtures small |
| RTD build grows huge if we execute notebooks on RTD | Hard rule: committed markdown only; Action owns execution |
| Marimo ↔ ipynb export fidelity | Smoke-check exported `.ipynb` opens in JupyterLab; document any marimo-only features to avoid in examples |
| Docstring campaign never ends | Bound to curated public allowlist; private code excluded; Phase 2 checklist |
| SEO / old Sphinx URLs break | `redirects` in Zensical/RTD for top pages (`getting_started/` → `getting-started/`, etc.) |
| Readers land on wrong version | RTD version switcher: v1.x branch vs latest/2.x clearly labelled |

---

## 7. Decisions (2026-07-17)

| # | Question | Decision |
|---|---|---|
| 1 | Hard cut vs dual Sphinx/Zensical | **Hard cut** to Zensical on `main`/2.0. RTD already keeps **v1.x branch** docs — no dual builder on the new stack. |
| 2 | Examples on RTD | **Committed markdown** only. A **GitHub Action** runs/renders examples (marimo → md + assets + ipynb export). |
| 3 | API surface in 2.0 | **Curated public story** only (allowlist). Deep trees stay out of Reference. |
| 4 | Normative specs | **Link out** to cellpy-core RTD; thin local wrappers — do not mirror. |
| 5 | Docstring enforcement | **Review-only for 2.0** (PR checklist + mkdocstrings preview). Lint later (§8). |
| 6 | README vs docs Home | **Short README → RTD.** Drop `adapted_readme`; RTD Home is the landing page. |
| + | Example authoring format | **Marimo** as source of truth; each example offers **download in both marimo and Jupyter (`.ipynb`)** formats. |

---

## 8. Future work (parked)

Not blocking the 2.0 docs cutover; record so they are not forgotten:

1. **Docstring lint on the public allowlist** — e.g. ruff/pydocstyle (or
   equivalent) in CI once the curated API set and Google-style campaign have
   settled; avoid noisy fails mid-migration.
2. **Marimo authoring guide** — short Development page: how to edit an
   example, run the Action locally, export rules (what to avoid so `.ipynb`
   export stays clean), and how figure assets are committed.
3. **Example gallery polish** — thumbnails / one-line “what you learn” cards
   on the Examples index (researcher browse, not a notebook dump).
4. **Interactive embeds (optional)** — if Zensical/marimo story improves,
   reconsider lightweight WASM or linked marimo apps for a *subset* of
   examples; default remains static committed markdown.
5. **Cross-link hygiene** — automated check that Specifications links to
   core RTD still resolve (versioned URLs preferred over `latest` where
   possible).
6. **i18n / secondary language** — out of scope unless demand appears;
   structure nav so a later translation layer is not blocked.
