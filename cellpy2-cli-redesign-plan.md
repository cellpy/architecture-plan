# Plan: CLI redesign (Typer + library-first commands)

**Date:** 2026-07-17
**Closes:** the CLI half of gap-analysis item **G11** (release plan §6 parked
"CLI inventory beyond `setup`"; config plan Step 5 already owns the
`cellpy setup` *config* migration, not the CLI framework).
**Direction (maintainer request):** migrate from Click to **Typer**; decouple
every command so the useful work is a plain Python function/method callable
from scripts and notebooks — the CLI is a thin adapter, not the home of the
logic.
**Foundation:** [config plan](cellpy2-configuration-and-parameters-plan.md)
(Step 5 `setup` / `migrate`), [batch redesign](cellpy2-batch-redesign-plan.md)
(`run` / journal recipes), [conventions](cellpy2-conventions-plan.md),
[docs plan](cellpy2-documentation-plan.md) (CLI reference pages),
cellpy-core's Typer precedent (`libs/local_fastnda/cli.py` already ships Typer
inside this repo).

---

## 1. Where the CLI lives today

One file, **`cellpy/cli.py` (~2 165 lines)**, Click-based, registered as
`[project.scripts] cellpy = "cellpy.cli:cli"`. Typer is already a dependency
(used by vendored `local_fastnda`); Click is the only framework the top-level
entry point uses.

### 1.1 Command inventory

| Command | Role | Logic today |
|---|---|---|
| `setup` (+ `setup migrate`) | Write/migrate user config, folders, env | ~700 lines mixed with Click prompts (`cli.py:150–970`) — config plan Step 5 owns the *config* side |
| `info` | Version, config location, params dump, `--check` | Thin; mostly echo helpers |
| `edit` | Open config / named file in `$EDITOR` | Path resolution + subprocess |
| `pull` | Download tests / examples / clone repo | GitHub blob download helpers (~200 lines) |
| `run` | Batch from journal / DB / folder | Calls into batch stack; large option surface |
| `new` | Cookiecutter-ish project from template | Template registry + cookiecutter (~250 lines) |
| `serve` | Launch Jupyter | Thin wrapper |
| `convert` | Legacy `.h5` → current cellpy file | Already extracted toward `cellpy_file/` (#449) — best existing example of "logic elsewhere, CLI thin" |

`_main` / `_main_pull` / `_cli_setup_interactive` / `_check_*` at the bottom are
dev/smoke entry points, not public commands.

### 1.2 What is wrong

1. **Logic trapped in Click callbacks.** Path resolution, config writing,
   template discovery, example download, batch orchestration — none of these
   are importable without going through Click contexts, `click.echo`, or
   `click.confirm`. A notebook that wants "run this journal like `cellpy run`"
   either shells out or copy-pastes private `_run_*` helpers.
2. **One megamodule.** Setup, batch run, templates, and GitHub downloads share
   a file; testing one command means importing all of them (and their optional
   deps).
3. **Click → Typer is decided** (maintainer). Staying on Click forever is not
   an option; the question is *how* to migrate without a big-bang rewrite of
   `setup`'s interactive UX.
4. **Inconsistent surfaces.** `convert` is already library-first;
   `setup`/`new`/`run` are not. Users learn two patterns.
5. **No typed options model.** Batch `run` alone has ~15 flags with stringly
   typed paths and overlapping force flags (`--raw` / `--cellpyfile`).

### 1.3 What is genuinely good (and must survive)

- The command *names* and rough UX: `setup`, `info`, `edit`, `pull`, `run`,
  `new`, `serve`, `convert` — researchers already know them.
- Silent / dry-run flags on setup and migrate (CI-friendly).
- `cellpy convert` as the pattern: public function in the domain package,
  CLI as a few lines of argument mapping.
- `info --check` as a health probe (keep; make it call a library `diagnose()`).

---

## 2. Design principles

1. **Library first, CLI second.** Every command's work lives in a public
   Python API (`cellpy.cli_api` or domain packages — see §3). The Typer
   layer only: parse argv → call function → print / set exit code.
2. **No Click/`typer.echo` inside library code.** Library functions return
   values / raise typed errors (`ConfigurationError`, etc. from the
   conventions plan); the CLI decides what to print. Prompts
   (`confirm`, `prompt`) stay in the CLI layer or take an injected
   `Prompt` protocol for tests.
3. **One framework: Typer.** Drop Click from `[project.dependencies]` once
   the entry point and tests no longer import it. Match the style already
   used in `local_fastnda` (`Annotated[...]`, `typer.Option`,
   `add_completion=False` unless we explicitly want shell completion).
4. **Stable command names; freer internals.** Subcommand spelling
   (`cellpy setup migrate`) stays. Package layout under `cellpy/cli/` may
   change freely.
5. **Scripts and notebooks are first-class clients.** Documented examples
   call `from cellpy.cli_api import run_journal, setup_config, …` — not
   `subprocess.run(["cellpy", …])`.
6. **Exit codes are part of the contract.** `0` success; `1` user/config
   error; `2` unexpected. Library functions do not `sys.exit`; the Typer
   layer maps exceptions → codes.

---

## 3. Proposed architecture

```
cellpy/
    cli/                      # Typer adapters only (thin)
        __init__.py           # app = typer.Typer(...); re-export for entry point
        app.py                # root app + callback (version, verbosity)
        setup_cmd.py
        info_cmd.py
        edit_cmd.py
        pull_cmd.py
        run_cmd.py
        new_cmd.py
        serve_cmd.py
        convert_cmd.py
        _prompt.py            # interactive Prompt protocol + Click/Typer impl
    cli_api/                  # OR: functions live in domain packages (preferred
                              #   when a natural home exists — see table below)
        __init__.py           # stable import facade for script authors
        setup.py              # write_config, migrate_config, ensure_dirs, …
        info.py               # diagnose, format_version_report, …
        pull.py               # pull_tests, pull_examples, clone_repo
        new.py                # create_project(template=..., …)
        run.py                # run_journal / run_batch — thin over cellpy.batch
        serve.py              # start_jupyter(...)
```

**Preferred placement rule:** if the logic belongs to an existing domain
package, put it there and re-export from `cli_api` for discoverability:

| Concern | Natural home | `cli_api` role |
|---|---|---|
| Config write / migrate / diagnose | `cellpy.config` (config plan) | re-export |
| File convert | `cellpy.readers.cellpy_file` (already) | re-export |
| Batch run from journal | `cellpy.batch` (batch redesign) | re-export / thin recipe |
| Templates / `new` | `cellpy.templates` (new small package) or `cli_api.new` | own or re-export |
| Example/test download | `cellpy.utils.example_data` (+ pull helpers) | re-export |
| Jupyter serve | `cli_api.serve` (no domain home needed) | own |

Entry point becomes:

```toml
[project.scripts]
cellpy = "cellpy.cli.app:app"
```

(Typer apps are callable; `typer.run` is not required when using a packaged
app object.)

### 3.1 Shape of a command (concrete)

```python
# cellpy/cli_api/run.py  — importable from scripts
def run_journal(
    path: Path,
    *,
    force_raw: bool = False,
    force_cellpy: bool = False,
    minimal: bool = False,
    nom_cap: float | None = None,
) -> BatchResult:
    """Run a batch journal end-to-end. Raises typed errors; no printing."""
    ...

# cellpy/cli/run_cmd.py  — Typer only
@app.command("run")
def run_cmd(
    name: Annotated[str, typer.Argument()] = "NONE",
    raw: bool = False,
    cellpyfile: bool = False,
    ...
) -> None:
    try:
        result = cli_api.run_journal(Path(name), force_raw=raw, ...)
    except ConfigurationError as e:
        typer.secho(str(e), fg=typer.colors.RED, err=True)
        raise typer.Exit(1) from e
    typer.echo(f"finished: {result.name}")
```

Script author:

```python
from cellpy.cli_api import run_journal
run_journal("my_batch.json", force_raw=True)
```

### 3.2 Interactive prompts

`setup` (and optionally `new`) need questions. Pattern:

```python
class Prompt(Protocol):
    def confirm(self, message: str, default: bool = True) -> bool: ...
    def ask(self, message: str, default: str | None = None) -> str: ...

class TyperPrompt:  # used by CLI
    ...

class SilentPrompt:  # --silent / library default: accept defaults, never block
    ...
```

Library `setup_config(..., prompt: Prompt | None = None)` uses `SilentPrompt`
when `None`. Tests inject a recording fake. No `input()` scattered in domain
code.

### 3.3 Verbosity and logging

Root callback sets log level (`-v` / `-q`) via `cellpy.log.setup` (conventions
plan — no import-time logging). Library code uses module loggers; CLI does not
re-implement logging.

### 3.4 What dies

| Today | Fate |
|---|---|
| Click as the CLI framework | removed after Typer parity |
| Business logic inside `@click.command` bodies | moved to `cli_api` / domain packages |
| `click.echo` / `click.confirm` in helpers | CLI layer or `Prompt` protocol only |
| Bottom-of-file `_main*` smoke scripts | `python -m cellpy.cli …` or pytest; delete or move to `dev/` |
| Ad-hoc `cli.add_command` registration | Typer sub-apps / `@app.command` |

---

## 4. Sizing

| Piece | Today | After |
|---|---|---|
| `cellpy/cli.py` monolith | ~2 165 | 0 (split) |
| `cellpy/cli/` (Typer adapters) | 0 | ~400–600 |
| `cli_api` + domain moves | (buried) | ~800–1 200 (mostly moved, not new) |
| Click dependency | yes | no |
| Typer dependency | yes (already) | yes (sole CLI framework) |

Net line count similar; testability and reuse are the win.

---

## 5. Migration plan

Trailing work relative to the flip (release plan §6), but **start as soon as
config Step 5's library surface is stable** — `setup` is the hardest command
and should not be rewritten twice.

### Phase 0 — inventory + facade (½–1 day)

1. Freeze the public command list and flags (table in §1.1); note which flags
   are dead / unused (grep docs, tests, maintainer notebooks).
2. Add `cellpy/cli_api/` with **re-exports / thin wrappers** that today's
   private helpers already implement — no behavior change. Entry point still
   Click.
3. Characterization tests: one CliRunner (or Typer's) smoke per command
   already covered; extend to assert the *library* function is what the
   command calls (monkeypatch spy).

### Phase 1 — extract library functions command-by-command (3–5 days)

Order (easy → hard, maximizing early script value):

1. `convert` — already mostly done; tidy re-export.
2. `info` / `serve` / `edit` — small, prove the pattern.
3. `pull` — move download helpers next to `example_data`.
4. `new` — template registry becomes a real module.
5. `run` — depends on batch redesign's public recipe surface; until then,
   wrap today's `_run_journal` without inventing a second batch API.
6. `setup` (+ `migrate`) — land *with* or *right after* config Step 5 so
   prompts talk to pydantic models, not `PathsClass` field lists.

Each PR: move logic → CLI calls it → tests hit both `cli_api` and the
command. No Typer yet (keeps diffs reviewable).

### Phase 2 — Click → Typer (2–3 days)

1. Stand up `cellpy/cli/app.py` with the same command names/flags.
2. Point `[project.scripts]` at the Typer app; keep a temporary
   `cellpy.cli:cli` shim that forwards for one minor if needed (prefer a
   clean cut — scripts entry is the only public path).
3. Port tests from Click's `CliRunner` to Typer's (`CliRunner` from
   `typer.testing`).
4. Remove Click from dependencies; grep-clean the repo.

### Phase 3 — docs + script recipes (1 day, with docs plan)

- CLI reference pages generated from Typer help (`typer` markdown export or
  hand-maintained short pages — decide in docs plan).
- "Automating cellpy" page: three copy-paste recipes using `cli_api`
  (setup silent, convert folder, run journal).
- Deprecate any documented `subprocess` patterns.

### Phase 4 — command trim (optional, 2.1)

After usage evidence: drop or hide rarely used flags; consider making
`pull --clone` a separate advanced command. Not blocking 2.0.

Total: **~1.5–2 weeks** spread across Stage 3 / trailing G11; Phases 0–1 can
start earlier on `main`/`v1.x` as behavior-preserving refactors.

---

## 6. Risks

| Risk | Mitigation |
|---|---|
| `setup` interactive UX regresses | Characterization tests on prompt sequences; `SilentPrompt` path is the CI gate; human dry-run checklist once |
| Double migration (Click extract, then Typer) feels wasteful | Phase 1 is the valuable part (library API); Phase 2 is mechanical and short *because* Phase 1 happened |
| `run` blocked on batch redesign | Wrap current helpers; swap the body when `cellpy.batch` recipes land — CLI flags stay |
| Typer version / Click leftover in transitive deps | Pin typer; assert `import click` fails in a dedicated test after Phase 2 |
| Users rely on private `_run_*` names | They were never public; document `cli_api` as the supported surface; no shim for `_`-names (conventions plan) |

---

## 7. Open questions for the maintainer

1. **Package name for the library facade:** `cellpy.cli_api` (explicit,
   script-friendly) vs putting everything only in domain packages with a
   thin `cellpy.cli` re-export list?
2. **Shell completion:** ship Typer completion (`--install-completion`) in
   2.0, or keep `add_completion=False` like `local_fastnda`?
3. **`cellpy run` vs batch Python API:** is the CLI allowed to expose flags
   that the batch library does not (convenience), or must every flag map
   1:1 to a `RunOptions` dataclass field?
4. **Template / cookiecutter future:** keep cookiecutter-based `new`, or
   slim to a few vendored templates copied by shutil (fewer deps)?
5. **Windows editor / Jupyter discovery:** keep today's heuristics, or
   require explicit `--editor` / `--executable` when auto-detect fails
   (fail loud — conventions plan)?
