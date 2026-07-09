# Plan: configuration and parameter handling in cellpy v2

**Date:** 2026-07-08
**Scope:** `cellpy/parameters/` (`prms.py`, `prmreader.py`, `.cellpy_prms_default.conf`),
the `.env_cellpy` secrets file, the `cellpy setup` CLI flow, and every consumer of
`prms.*` in the package (383 usages across 34 files). Companion doc:
`cellpy-file-loading-refactor-plan.md` (the `_cellpyfile_*` constants overlap;
cross-referenced below).
**Goal:** Replace the home-grown prms system with a layered, typed, validated
configuration stack for cellpy v2 — while keeping cellpy-core configuration-free
(*core owns shapes and tools; the consumer owns content and policy*), and giving the
v1 API a soft landing.

---

## 1. Current state — inventory

### 1.1 The three-part home-grown system

**(a) `prms.py` — global mutable singletons.** Plain dataclasses instantiated at
import time (`Paths`, `FileNames`, `Reader`, `Db`, `DbCols`, `CellInfo`, `Materials`,
`Batch`) plus an `Instruments` class whose per-instrument settings are untyped
`box.Box` dicts (`Arbin`, `Maccor`, `Neware`, `Batmo`). Below the classes live ~60
module-level "secret/internal" underscore constants: the cellpy-file format spec
(`_cellpyfile_*`), GitHub/template URLs, ODBC knobs, and even mutable runtime state
(`_globals_status`, `_globals_errors`).

**(b) `prmreader.py` — hand-rolled load/save.** ruamel-YAML round-trip of a
`.cellpy_prms_<username>.conf` file discovered in the user's home dir
(`_get_prm_file` with a glob + ordered search), `_update_prms()` walking the dict with
special-case branches (Paths → `OtherPath`/`pathlib.Path` coercion + resolve;
dicts → `box.Box` merge), and `_load_env_file()` piping `.env_cellpy` through
`python-dotenv`. Secrets (`CELLPY_PASSWORD`, `CELLPY_KEY_FILENAME`, `CELLPY_HOST`,
`CELLPY_USER`) are then read via raw `os.getenv` in `internals/otherpath.py` and
`internals/connections.py`.

**(c) `cellpy setup` (cli.py, 44 prms references).** Interactively writes the user
conf file and creates the folder structure; `_update_paths` mirrors `PathsClass`
field-by-field.

### 1.2 How it is wired and consumed

- `cellpy/__init__.py:37–38` runs `prmreader.initialize()` **at import time** — every
  `import cellpy` reads the user's config file and `.env` file. The in-code TODO
  already marks this for removal in v2.0.
- Consumers read the singletons at call time (`prms.Reader.cycle_mode`,
  `prms.Instruments.Arbin.max_res_filesize`, `prms.Paths.rawdatadir`, …) — 383 sites.
- **Mutation is a public API.** Documented user pattern: set
  `prms.Reader.cycle_mode = "cathode"` or `prms.Instruments.tester = "arbin"` before
  calling `cellpy.get`. Tests and batch code do the same.
- Instrument YAML definition files (`instrumentdir`, `local_instrument.py`,
  `custom_instrument_definitions_file`) form a second, separate config channel for
  instrument formats.

### 1.3 Known pain points (from the code's own comments and structure)

1. **Global mutable state.** Session config, per-cell science defaults, app constants,
   and runtime status flags share one namespace of writable module globals. Two
   consequences the codebase already exhibits: test pollution (config leaks between
   tests unless carefully reset) and no way to run two differently-configured sessions
   in one process. `arbin_res.py` even treats `prms.Reader.limit_loaded_cycles` as a
   *different type* (a `[from, to]` list) than its declared `Optional[int]`.
2. **No validation.** `_update_prms` silently `setattr`s whatever the YAML contains;
   an unsupported section is a mere `logging.info("not-supported prm")`. A typo in the
   conf file becomes a wrong-type attribute at runtime.
3. **N-way manual sync.** The header comments in `prms.py` spell it out: adding one
   parameter requires touching `prms.py`, `.cellpy_prms_default.conf`,
   `internal_settings.py`, `cli.py`, and two test files. `DbColsClass` carries a
   four-step instruction list.
4. **Mixed concerns in one file.** The cellpy-file format spec (`_cellpyfile_*`),
   template registry, and example-data URLs are *constants*, not configuration; the
   `CellInfo`/`Materials` sections are *per-cell science metadata defaults*, not
   session config. All live in `prms.py`.
5. **Fragile path/env handling.** `env_file` location confusion is immortalized in
   `prmreader.get_env_file_name()`'s "WHY ARE THEY NOT IN THE SAME DIRECTORY?"
   comment; the OtherPath/`db_filename` special-cases in `_update_prms` are branch-y
   and untested edge ground.
6. **`box.Box` for instruments** — untyped, merge-by-dict, flagged in `prms.py` with
   "maybe replace later using e.g. pydantic".
7. **Import-time side effects** — reading files at `import cellpy` breaks isolated /
   embedded use and is already slated for v2 removal.

---

## 2. Requirements for v2

1. **Layered precedence, resolved once, inspectable.** built-in defaults → user
   config file → project-local config file → environment variables (+ `.env`) →
   runtime overrides (explicit function arguments always win). Users must be able to
   ask *where a value came from*.
2. **Typed and validated at the boundary.** A bad config file fails loudly at load
   time with the offending key and file named — never as a silent wrong-type
   attribute.
3. **Single source of truth = the model definitions.** The default config file is
   *generated* from the models (`cellpy setup` writes it; docs render it); adding a
   field is a one-file change.
4. **Four kinds of "parameters", four different homes:**
   | Kind | Examples | v2 home |
   |---|---|---|
   | Application config | paths, db connection, file-name patterns, batch/plot prefs, instrument I/O knobs | the new config stack (this plan) |
   | Secrets | SSH password/key, DB credentials | environment / `.env` only — never in the config file, never echoed |
   | Library/engine parameters | step-detection thresholds (`CellpyLimits`), interpolation steps, `TestMode` | explicit function/constructor arguments into cellpy-core; config may *supply* them at the app boundary |
   | Per-cell science metadata | mass, area, nom_cap, cell_type, electrolyte… (`CellInfo`, `Materials`) | `cellpycore.metadata` models (`CellMeta`/`TestMeta`); config holds only *defaults* used when constructing them |
5. **cellpy-core stays configuration-free.** Core never reads files or environment
   variables. It already has the right shapes: `BaseSettings`/`DictLikeClass`
   (`settings_base.py`), `CellpyLimits`, typed `config.Cols`, `TestMode` StrEnum.
   The consumer (cellpy v2) resolves configuration and passes values in.
6. **No import-time I/O.** Config loads lazily on first access or on explicit
   `cellpy.init()`; importing the library is side-effect-free.

---

## 3. Target architecture

### 3.1 Technology: pydantic-settings (pydantic v2), TOML config file

- **pydantic-settings** gives layered sources (init kwargs > env > `.env` > file >
  defaults), `CELLPY_`-prefixed env vars with `__` nesting
  (`CELLPY_READER__CYCLE_MODE=cathode`), `SecretStr` for credentials, strict typing
  with coercion, and `validate_assignment=True` so the *mutable-session* ergonomics
  survive but validated. `prms.py`'s own comment ("replace later using e.g.
  pydantic") points here; pydantic v2 is ubiquitous in the Jupyter/scientific stack
  cellpy users already run.
- **TOML** (`cellpy.toml`) replaces the YAML `.conf`: stdlib-readable (`tomllib`),
  comment-friendly, the format users now expect from `pyproject.toml`-era tooling.
  The old YAML format stays *readable* through a migration shim for the v2.0 cycle
  (ruamel is already a dependency; drop it when the shim goes).
- **platformdirs** for locations: user config at
  `platformdirs.user_config_dir("cellpy")/cellpy.toml`, ending the home-dir glob and
  the env-file location confusion. A project-local `./cellpy.toml` (walk-up from cwd,
  like git) layers on top — this is new capability the current system lacks and
  batch-processing users regularly ask for (per-project data dirs).
- Dependency budget: pydantic-settings + platformdirs in **cellpy only**. cellpy-core
  gains nothing (requirement 5). Net for cellpy: −python-box, −ruamel (after
  migration window), −dotenv (pydantic-settings handles `.env`).

### 3.2 The model tree (`cellpy/config/models.py`)

Mirrors today's sections so the migration is mechanical, with renames only where a
section changes *kind*:

```python
class CellpyConfig(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="CELLPY_", env_nested_delimiter="__",
        validate_assignment=True, toml_file=...)

    paths: Paths                 # ex PathsClass; OtherPath via Annotated custom type
    file_names: FileNames        # ex FileNamesClass
    reader: Reader               # ex ReaderClass (minus per-cell leftovers, see §3.4)
    db: Db                       # ex DbClass + DbColsClass (simple-excel reader only)
    batch: Batch                 # ex BatchClass
    instruments: Instruments     # typed models replacing the box.Box dicts
    defaults: ScienceDefaults    # ex CellInfoClass + MaterialsClass — *defaults for
                                 #    metadata construction*, clearly labeled as such
    units: Units                 # NEW, no prms ancestor (added 2026-07-09, cross-check
                                 #    vs unit-handling-cellpy2-plan.md): session unit
                                 #    policy (cellpy_units overrides, e.g. charge="Ah");
                                 #    validated against cellpycore.units.CellpyUnits keys
    secrets: Secrets             # SecretStr fields; env/.env-only (excluded from file dump)
```

- `Instruments`: one pydantic model per instrument (`ArbinConfig`,
  `MaccorConfig`, …) with an `extra="allow"` escape hatch so plugin instruments can
  attach settings without core changes; SQL credentials move to `Secrets`/env (today
  `SQL_PWD: "ChangeMe123"` sits in plain text in `prms.py`).
- `OtherPath` fields use an `Annotated` type with a pydantic validator wrapping
  today's `OtherPath(str(v))` + resolve logic — the `_update_prms` special-case
  becomes a declared type.
- **What does *not* move in:** the `_cellpyfile_*` format spec (→
  `readers/cellpy_file/format.py`, Step 1 of the companion plan), template registry
  and example-data URLs (→ their owning modules as plain constants),
  `_globals_status`/`_globals_errors` runtime state (→ deleted or made explicit
  return values). Constants are not configuration.
- `ScienceDefaults` values (`default_mass`, `default_nom_cap`, …) are **in cellpy
  units by convention** — state it on the model docstring/fields. They feed the
  metadata plan's `MetaResolver` as the lowest layer; without the convention a site
  config in `g` would silently disagree with a loader supplying `mg`
  *(added 2026-07-09, cross-check vs unit-handling-cellpy2-plan.md)*.
- `Units` is the config home the unit plan's Phase 5 assumes ("cellpy_units
  initialised from user config exactly once"); legacy prms has **no** units section,
  so this is new capability, and `override(units={...})` gives the scoped unit
  overrides notebooks otherwise hack in by mutating `cellpy_units` at runtime.

### 3.3 Access pattern and lifecycle (`cellpy/config/__init__.py`)

```python
import cellpy

cellpy.config.reader.cycle_mode          # lazy singleton, loads layers on first touch
cellpy.config.reader.cycle_mode = "cathode"   # still works — now validated

with cellpy.config.override(reader={"cycle_mode": "cathode"}):
    c = cellpy.get(...)                  # scoped override, auto-restored (test-friendly)

cellpy.config.sources()                  # provenance: value → default/file/env/runtime
cellpy.config.reload(profile="myproject")  # explicit re-read; profiles replace the
                                           # .cellpy_prms_<user>.conf-per-user scheme
```

- The singleton is created lazily (PEP 562 `__getattr__` on the package) — no
  import-time file reads. `cellpy.init()` remains as an explicit eager initializer.
- `override()` is the sanctioned scoped-mutation tool; a pytest fixture
  (`isolated_config`) built on it ships with cellpy so downstream tests stop leaking
  global state.
- Function-level kwargs (e.g. `cellpy.get(..., cycle_mode=...)`) keep absolute
  precedence — resolved at the call site, config only consulted for unset arguments.

### 3.4 The core seam (what crosses into cellpy-core)

Config values are **resolved at the boundary and passed as values**:

- `CellpyLimits` (step-detection thresholds): constructed by cellpy from
  `config.reader` values, handed to core's step engine.
- `cycle_mode` string → `cellpycore.config.TestMode` enum at the boundary (the
  mapping and its polarity trap are already documented on the enum).
- Interpolation steps, `max_raw_files_to_merge` etc.: plain arguments.
- `ScienceDefaults` feed `cellpycore.metadata.CellMeta`/`TestMeta` construction in
  the app layer; core never sees the config object.

This keeps core testable with literals and honors the migration doc's boundary
(`cellpy-core-migration.md` §4).

### 3.5 v1 compatibility shim

`cellpy.parameters.prms` becomes a thin module with `__getattr__` mapping the old
names onto the new config (`prms.Reader` → `cellpy.config.reader`, attribute writes
forwarded, `DeprecationWarning` once per name). The 383 internal call sites are
migrated mechanically (regex-able: `prms\.(\w+)\.(\w+)` → `config.<section>.<attr>`);
the shim exists for *user* code and notebooks, not for cellpy itself. Lifetime: the
v2.0 cycle, removal in v2.1.

---

## 4. Migration plan — steps, each suite-green

### Step 0 — Characterization tests
Lock in current behavior before touching it: config-file round-trip (`test_prms.py`
exists — extend), precedence of file vs. runtime mutation, OtherPath coercion in
`Paths`, `.env` secret pickup by `OtherPath`, `cellpy setup` file creation. Add an
inventory test that asserts the exact set of (section, field, default) triples —
this becomes the parity contract for the new models.

### Step 1 — Purge non-config constants from `prms.py`
Move `_cellpyfile_*` to `readers/cellpy_file/format.py` (companion plan Step 1 —
do it once, in whichever plan executes first), template/URL constants to their
modules, delete `_globals_*` in favor of explicit returns. Alias from `prms` during
the window. Small, independent, immediately shrinks the problem.

### Step 2 — Build the new stack in parallel
`cellpy/config/` package: models (§3.2), layered loader (§3.3), TOML I/O,
YAML→TOML converter, provenance tracking. Parity test from Step 0 asserts the new
defaults equal the old ones field-by-field. Nothing else imports it yet.

### Step 3 — Swap the engine underneath `prms`
Re-point the `prms` module onto the new config via the shim (§3.5); delete
`_update_prms`/`_pack_prms`/file-glob discovery. All 383 internal call sites still
work unchanged through the shim — this is the flag-day step, kept safe by Steps 0+2.
`initialize()` becomes a call into `config.reload()` (still invoked from
`__init__.py` for now — import-time behavior unchanged in this step).

### Step 4 — Migrate internal call sites + kill import-time init
Mechanical rewrite `prms.X.y` → `cellpy.config.x.y` across the package (mirrors the
method used in `hardcoded-column-headers-report.md`); remove `init()` from
`__init__.py` (the v2.0 TODO at `__init__.py:35`); lazy loading takes over. The shim
now serves only external code and starts warning.

### Step 5 — `cellpy setup` rewrite + migration UX
`cellpy setup` generates `cellpy.toml` from the models; detects an existing
`.cellpy_prms_<user>.conf` and offers conversion (`cellpy setup migrate`); folder
creation logic unchanged. `cellpy info --config` prints resolved values with
provenance (replaces `prmreader.info()`).

### Step 6 — Boundary hardening (with the core seam work)
Secrets → `Secrets` model consumed by `otherpath.py`/`connections.py` (drop raw
`os.getenv` scatter); `cycle_mode`→`TestMode` conversion at the core boundary;
`CellpyLimits` construction from config. Timed with the jepegit/cellpy#377 seam PRs.

### Step 7 — Docs, deprecation, removal schedule
Configuration docs page generated from the models; `prms` shim documented as
deprecated with the v2.1 removal date; HISTORY.md migration guide (old key → new key
table, also generated).

Steps 1, 2, 3, 4, 5 are separate PRs; 6 rides with the core-seam work.

---

## 5. Decisions taken

1. **pydantic-settings, not hand-rolled dataclasses and not dynaconf.** Hand-rolled
   is what we have — the pain list in §1.3 is its report card. dynaconf brings its
   own global-object philosophy and less typing. pydantic v2 gives validation,
   env/.env/TOML sources, and `SecretStr` for the price of one well-maintained,
   already-ubiquitous dependency — and `prms.py` has pointed at it since the comment
   was written. Confined to the cellpy app layer; core stays dependency-lean.
2. **TOML + platformdirs; YAML readable through v2.0 only.** New file
   `cellpy.toml` in the platform config dir plus optional project-local overlay;
   automatic one-time conversion in `cellpy setup migrate`. Revisit trigger: none —
   the converter makes this cheap for users.
3. **Mutable-but-validated session config, with `override()` as the blessed scoped
   mechanism.** A frozen config would be cleaner but breaks the single most-used
   pattern in notebooks (`prms.Reader.x = y`). `validate_assignment=True` keeps the
   ergonomics and closes the type-safety hole; `override()` + the pytest fixture
   give tests and libraries a leak-free path.
4. **Per-cell science parameters leave configuration.** `CellInfo`/`Materials`
   survive only as a `defaults:` section consumed when constructing
   `cellpycore.metadata` records. The record on the cell, not the global, is the
   truth — this aligns with the metadata scaffolding already landed in core.
5. **Secrets are env-only.** `Secrets` model with `SecretStr`, populated exclusively
   from environment/`.env`; excluded from every file dump and `info` printout. The
   plain-text `SQL_PWD` default in `prms.py` is removed, not migrated.
6. **cellpy-core gets no config system.** Reaffirmed; enforced by a lint/CI check in
   core (no `os.environ`, no file reads outside `metadata.io`).

---

## 5b. Addendum: OtherPath decision (G7, added 2026-07-09)

The gap analysis flagged `internals/otherpath.py` (ssh/scp-backed remote paths +
`.env` credentials) as undecided for cellpy 2 while three plans depend on it (this
plan's `Secrets`/`Paths`, the file-loading plan's external-file temp copy, every
loader path). **Decision proposed: keep OtherPath in 2.0, wrapped, and decide
fsspec later.**

- Keep the existing implementation (it works and is tested via
  `test_otherpaths.py`); wrap it as the pydantic `Annotated` type from §3.2 and
  route its credential reads through the `Secrets` model (Step 6) — that removes the
  raw `os.getenv` scatter without a rewrite.
- Centralize the "external file → local temp copy" behavior in exactly one boundary
  function inside `cellpy/readers/cellpy_file/` (the file plan already requires
  this) and reuse it for raw-file loading, so a future fsspec swap touches one seam.
- Revisit trigger for fsspec/universal-pathlib: a second remote scheme beyond ssh
  (s3, http) is actually requested. Do **not** hard-code OtherPath semantics into the
  loader-port configurations (G2) beyond "a path-like the framework resolves" —
  that keeps G7 cheap to reopen.

## 6. Risks and mitigations

| Risk | Mitigation |
|---|---|
| The `prms.Reader.limit_loaded_cycles` type lie (int vs `[from, to]` list in `arbin_res.py`) breaks under validation | Fix the field to a proper `int \| tuple[int, int] \| None` union in Step 2 — validation *finding* these is the point; audit all fields against actual call-site usage during model writing |
| User notebooks mutate `prms.*` everywhere | Shim keeps every documented pattern working through v2.0 with warnings; migration table in docs is generated, not hand-written |
| OtherPath as a pydantic type (custom scheme/host parsing) | Dedicated `Annotated` type with tests ported from `test_otherpaths.py`; reuse existing `OtherPath` code, wrap don't rewrite |
| box.Box merge semantics for instrument dicts (users add ad-hoc keys) | `extra="allow"` on instrument models preserves the escape hatch; typed fields cover the documented knobs |
| Import-time init removal changes behavior for scripts relying on `import cellpy` side effects | Lazy load on first config access means values are identical by the time anything reads them; only *timing* of file reads changes — call out in HISTORY.md |
| pydantic version conflicts in user environments | Require `pydantic>=2`; v1-pinned environments are rare and shrinking; no `pydantic.v1` shims |
| CLI `setup` tests (`test_cli_setup_interactive`, `NUMBER_OF_DIRS`) are tightly coupled to `PathsClass` | They migrate in Step 5 with the CLI itself; parity test from Step 0 guards the field set |
| Parallel work: companion file-loading plan also touches `prms._cellpyfile_*` | Single shared Step (this plan's Step 1 = that plan's Step 1); whoever lands first does it, the other rebases |
| Parallel work: unit plan expects a `units:` config section (no prms ancestor to migrate) *(added 2026-07-09)* | `Units` model added in §3.2; keys validated against `cellpycore.units.CellpyUnits`; coordinate with unit plan Phase 5 |

---

## 7. Estimated effort

| Step | Size | Notes |
|---|---|---|
| 0 Characterization + inventory tests | 1 day | Parity contract is the keystone |
| 1 Constants purge | 0.5 day | Shared with file-loading plan |
| 2 New config stack | 2–3 days | Models are mechanical; loader/provenance is the real work |
| 3 Shim swap | 1 day | Flag-day PR, small diff, big test run |
| 4 Internal call-site migration | 1–2 days | Mostly mechanical (383 sites); instruments need care |
| 5 CLI setup + migrate | 1–2 days | Includes conversion UX |
| 6 Boundary hardening | 1 day | Timed with core-seam PRs |
| 7 Docs + deprecation | 1 day | Docs generated from models |

Total ≈ 8–11 focused days across 6–7 PRs, each independently shippable. Steps 0–2
can start immediately; Step 6 waits for the cellpy-core seam (jepegit/cellpy#377).
