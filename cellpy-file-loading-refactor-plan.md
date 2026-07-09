# Refactoring plan: extract cellpy-file (HDF5) loading out of `CellpyCell`

**Date:** 2026-07-08
**Scope:** `cellpy/readers/cellreader.py` (7,440 lines) and every other place in the
`cellpy` package that reads or writes the cellpy file format (HDF5).
**Goal:** Isolate all cellpy-file I/O into a dedicated, stateless module with a small
functional API, so that (a) `CellpyCell` shrinks toward a thin orchestrator, and
(b) cellpy v2 can plug persistence in behind a clean seam — consistent with the
cellpy-core boundary decision: *core owns shapes and tools; the consumer owns content
and heavy persistence* (see
`cellpy-core/.issueflows/04-designs-and-guides/cellpy-core-integration-into-cellpy.md`).

This is a **behavior-preserving refactor**. No file-format changes (stay at
`CELLPY_FILE_VERSION = 8`), no dependency changes. Format evolution (v9, non-HDF5
backends, BDF) becomes *possible* after this, but is explicitly out of scope.

---

## 1. Current state — inventory

### 1.1 The code that must move (all in `cellreader.py`, methods on `CellpyCell`)

Reading path:

| Method | Lines (approx.) | Role |
|---|---|---|
| `load()` | 1456–1521 | Public entry point; wraps `_load_hdf5`, assigns `self.data`, bookkeeping |
| `_load_hdf5()` | 1763–1827 | Version gate + dispatch (too new / too old / old-but-accepted / current) |
| `_get_cellpy_file_version()` | 1734–1760 | Opens store, reads `/info` meta, extracts version |
| `_load_hdf5_current_version()` | 1830–1882 | v8 reader: orchestrates the `_extract_*` helpers |
| `_load_hdf5_v7()` / `_v6()` / `_v5()` | 1885–2047 | Per-version readers (different key layouts / meta handling) |
| `_load_old_hdf5()` | 2048–2066 | Old-version dispatch (v<5, 5, 6, 7) |
| `_load_old_hdf5_v3_to_v4()` | 2069–2125 | Oldest supported layout (`/dfdata`, `/dfsummary`, `/step_table`, `/fidtable`) |
| `_create_initial_data_set_from_cellpy_file()` | 2128–2164 | Creates `Data`, reads meta table(s) |
| `_check_keys_in_cellpy_file()` | 2168–2181 | Validates required store keys (static) |
| `_hdf5_locate_data_points_from_max_cycle_number()` | 2184–2200 | Selector support |
| `_hdf5_cycle_filter()` | 2203–2218 | Builds `where=` clauses from **mutable self-state** |
| `_unpack_selector()` | 2221–2225 | Stub (selector normalization, unimplemented) |
| `_extract_summary_from_cellpy_file()` | 2228–2264 | Reads summary; **sets `self.limit_loaded_cycles` / `self.limit_data_points` as side effects** |
| `_extract_raw_from_cellpy_file()` | 2267–2284 | Reads raw; depends on the side effect above |
| `_extract_steps_from_cellpy_file()` | 2286–2310 | Reads steps; same dependency |
| `_extract_fids_from_cellpy_file()` | 2313–2333 | Reads fid table |
| `_extract_meta_from_cellpy_file()` | 2336–2377 | v8 meta → `Data.meta_common` / `meta_test_dependent` |
| `_extract_meta_from_old_cellpy_file_max_v7()` | 2380–2427 | Legacy meta unpacking + unit-label conversion |
| `_extract_from_meta_dictionary()` | 2431–2442 | Small static helper |
| `_convert2fid_list()` | 2528–~2560 | fid table → `FileID` objects (static) |

Writing path:

| Method | Lines (approx.) | Role |
|---|---|---|
| `save()` | 1523–1659 | Public entry point; policy checks (overwrite, ensure step/summary) + format constants |
| `_save_to_hdf5()` | 1661–1731 | Puts raw/summary/meta/fid/steps into store; **returns an open `HDFStore`** that the caller closes in a `finally` |
| `_create_infotable()` | 2445–2479 | `Data` meta → DataFrames for storage |
| `_convert2fid_table()` | 2482–2524 | `FileID` objects → fid table (static) |
| `_fix_dtype_step_table()` | 3675 | dtype coercion retry when `store.put` raises `TypeError` |

Freshness / identity checking (reads the cellpy file out-of-band):

| Method | Lines | Role |
|---|---|---|
| `check_file_ids()` | 957 | Public; compares raw-file fids against cellpy-file fids |
| `_check_cellpy_file()` | 1041–1105 | Opens store directly, reads fid table; handles `OtherPath.is_external` by copying to a temp dir |
| `_check_HDFStore_available()` | 1033 | Import sanity check |

Sixteen of these already carry the comment
`# TODO @jepe: move this to its own module (e.g. as a cellpy-loader in instruments?)` —
this plan is the execution of that TODO (with a different landing spot than
`instruments/`, see §3.1).

### 1.2 Format knowledge scattered outside `cellreader.py`

- **`cellpy/parameters/prms.py:391–408`** — the de-facto format spec:
  `_cellpyfile_root = "CellpyData"`, key names (`/raw`, `/steps`, `/summary`, `/fid`,
  `/info`, `/info_test_dependent`), unit/limit key prefixes, compression, and the
  pandas `put` formats (`table` vs `fixed`).
- **`cellpy/parameters/internal_settings.py:12–21`** — `CELLPY_FILE_VERSION = 8`,
  `MINIMUM_CELLPY_FILE_VERSION = 4`, `PICKLE_PROTOCOL = 4`.
- **`cellpy/parameters/legacy/update_headers.py`** — `rename_raw_columns`,
  `rename_summary_columns`, `rename_step_columns`, `rename_fid_columns` used by the
  old-version readers.

### 1.3 Out-of-band readers that bypass `CellpyCell.load()` entirely

These read cellpy files with raw `pd.HDFStore` calls and duplicated/hard-coded layout
knowledge — they must end up going through the new module:

- **`cellpy/utils/batch_tools/batch_helpers.py:25–54`** — `look_up_and_get()`
  (batch "link" mode): opens the store directly, hard-codes `root = "/CellpyData"`,
  re-implements the max-cycle selector. Called from
  `batch_tools/batch_core.py:339` and `batch_experiments.py`.
- **`cellreader.py:_check_cellpy_file()`** (see §1.1) — direct fid-table read with its
  own OtherPath/temp-copy handling.
- **`cellpy/readers/filefinder.py`** — only deals with file *names* (`.h5` extension),
  no format knowledge; unaffected apart from possibly importing the extension constant.
- **`cellpy/utils/collectors.py` / `example_data.py`** — touch `.h5` files but not the
  cellpy-file layout (own keys / just downloads); out of scope.

### 1.4 Consumers of the loading API (must keep working unchanged)

- `cellpy.get` → `cellreader.py:7035` `cellpy_instance.load(...)`
- `CellpyCell.loadcell()` (deprecated) and `_dev_update_loadcell()` → `self.load(...)`
- `utils/batch_tools/batch_core.py:357` → `cellpy_data.load(file_name, self.parent_level, selector=selector)`
- `utils/ica.py:1105` and assorted tests → `cell.load(...)`
- Test assets: `testdata/hdf5/20160805_test001_45_cc_v{0,4,5,6,7,8}.h5` +
  `tests/test_cell_readers.py` etc. — excellent characterization material already in-tree.

---

## 2. Why this refactor (problems with the current design)

1. **God-object entanglement.** File-format parsing lives on the same 7,400-line class
   as step/summary computation, unit handling, and plotting helpers. cellpy-core will
   take over the compute half (`Data` ownership moves to `cellpycore` per the seam plan);
   the I/O half needs its own home or `CellpyCell` cannot shrink.
2. **Hidden mutable state as a data channel.** `_extract_summary_from_cellpy_file()`
   sets `self.limit_loaded_cycles` and `self.limit_data_points`; `_hdf5_cycle_filter()`
   and the raw/steps extractors silently read them. Extraction order is therefore
   load-bearing (summary **must** be read before raw/steps), and a `selector` on one
   load leaks state into the next load on the same instance
   (`self.limit_loaded_cycles = None` is only reset on the no-selector branch).
3. **Broken ownership of the store handle.** `_save_to_hdf5()` returns an *open*
   `HDFStore` for `save()` to close in a `finally` — an accident waiting to happen.
4. **Duplicated, drift-prone layout knowledge.** `batch_helpers.look_up_and_get()`
   hard-codes `"/CellpyData"` and re-implements cycle filtering; `_load_hdf5_v7()` has
   its own copies of key names as kwarg defaults.
5. **Error-handling smells that block diagnosis.** `load()` catches bare
   `AttributeError` and turns *any* attribute bug anywhere in the read path into
   "file version not supported"; `_check_keys_in_cellpy_file` raises bare
   `Exception("OH MY GOD! ...")`; several `except Exception` + `warnings.warn` blocks
   swallow real corruption.
6. **No format seam.** `save(cellpy_file_format="hdf5")` pretends pluggability but
   hard-locks to HDF5, and the pickle-protocol context manager + suppressed
   `PerformanceWarning` (object-dtype pickling in the meta tables) are format decisions
   buried in class methods. cellpy-core deliberately stubs
   `metadata.io.load_archive/save_archive` with `NotImplementedError`, expecting the
   consumer (cellpy v2) to provide exactly this adapter.

---

## 3. Target architecture

### 3.1 Landing spot: a dedicated subpackage, **not** `readers/instruments/`

The in-code TODOs suggest "a cellpy-loader in instruments". Recommendation: **do not**
put it there. The instruments framework (`instruments/base.py`: `BaseLoader`,
`AutoLoader`, `TxtLoader`) is built for *raw cycler exports*: it produces a bare `Data`
that then goes through step/summary generation, and it participates in
instrument-selection config. A cellpy file is different in kind — it is our own
serialization of the *complete* object (raw + steps + summary + meta + fids + format
version), it has version-gate semantics, and it also needs a **writer**, which the
instruments API has no concept of. Forcing it into `BaseLoader` buys a registry we
don't need and costs an awkward fit.

Create instead:

```
cellpy/readers/cellpy_file/
    __init__.py        # public API (functions below)
    format.py          # CellpyFileFormat spec + version constants (single source of truth)
    hdf5.py            # pandas/pytables backend: open/close, select, put, key checks
    read.py            # version gate + v8 reader (orchestration, Data assembly)
    legacy_read.py     # v3–v7 readers + rename-columns wiring (imports parameters/legacy)
    write.py           # save path: infotable/fid-table building + store writing
    selectors.py       # LoadSelector / LoadResult dataclasses (explicit, immutable)
    fids.py            # FileID <-> fid-table conversion + fid-based freshness check
```

### 3.2 Public API (module functions, no class state)

```python
# cellpy/readers/cellpy_file/__init__.py

def load(path, *, accept_old=True, selector=None, parent_level=None) -> LoadResult: ...
def save(data, path, *, overwrite=True, format_spec=None) -> None: ...
def get_version(path) -> int: ...
def read_fid_table(path) -> tuple[list[FileID], list[int]]: ...   # for check_file_ids
def read_table(path, table_name, *, max_cycle=None) -> pd.DataFrame: ...  # for batch link mode
```

Key design rules:

- **Operates on `Data`, never on `CellpyCell`.** `load()` returns a `LoadResult`;
  `save()` takes a `Data`. This is what makes the cellpy-core seam clean later: when
  `Data` ownership moves into `cellpycore`, the I/O module doesn't care.
- **`LoadResult` replaces the self-state side channel.**
  ```python
  @dataclass
  class LoadResult:
      data: Data                      # raw/steps/summary/meta populated
      file_version: int
      limit_data_points: int | None   # was: self.limit_data_points (side effect)
      limit_loaded_cycles: int | None # was: self.limit_loaded_cycles (side effect)
  ```
  `CellpyCell.load()` copies these onto itself for backward compatibility; nothing
  inside the module mutates caller state.
- **`LoadSelector` is normalized once** (finishing the `_unpack_selector` stub): parse
  `{"max_cycle": ...}` into a typed object at the API boundary; extractors receive it
  explicitly instead of consulting `self`.
- **`format.py` owns the layout.** A frozen dataclass `CellpyFileFormat` with
  `root`, `raw_dir`, `step_dir`, `summary_dir`, `fid_dir`, `common_meta_dir`,
  `test_dependent_meta_dir`, unit/limit prefixes, `complib`, `complevel`, and the
  per-table pandas formats. One instance per historical file version (v3/4, v5, v6,
  v7, v8) — this replaces both the `prms._cellpyfile_*` globals and the kwarg-default
  copies inside `_load_hdf5_v7`. `prms._cellpyfile_*` stay as aliases reading from
  `format.py` during a deprecation window (they are underscore-private, but
  `batch_helpers` and possibly user scripts touch them).
- **Version dispatch is a registry**, not an if/elif ladder:
  `_READERS: dict[int, Callable[[Path, LoadSelector], LoadResult]]` in `read.py` /
  `legacy_read.py`. Adding v9 later = adding an entry.
- **The writer owns its store lifecycle** (`with pd.HDFStore(...) as store:` inside
  `write.py`); no open handles cross function boundaries. The pickle-protocol context
  manager and the `PerformanceWarning` suppression move inside `hdf5.py` where the
  decision belongs.
- **Remote files handled at the boundary.** `load()` accepts `str | Path | OtherPath`;
  if `OtherPath.is_external`, copy to temp *once* at the top (the logic currently only
  present in `_check_cellpy_file`, line 1059–1063 — note that `_load_hdf5`'s
  `os.path.isfile()` check does **not** handle external paths today, so this refactor
  also fixes an inconsistency; keep the fix behavior-compatible: local paths behave
  identically).
- **Typed exceptions.** Replace the bare `Exception` in `_check_keys_in_cellpy_file`
  with `CorruptCellpyFile` (new, subclass of `IOError`) and keep raising
  `WrongFileVersion` for version-gate failures. `CellpyCell.load()` keeps its current
  lenient logging-warning behavior at *its* level so the public contract is unchanged.

### 3.3 What stays on `CellpyCell`

Thin wrappers only — public API stability for the v1.x line:

```python
def load(self, cellpy_file, parent_level=None, return_cls=True, accept_old=True,
         selector=None, **kwargs):
    result = cellpy_file_module.load(cellpy_file, accept_old=accept_old,
                                     selector=selector, parent_level=parent_level)
    self.data = result.data
    self.limit_data_points = result.limit_data_points
    self.limit_loaded_cycles = result.limit_loaded_cycles
    ...bookkeeping (name, last_uploaded_from, timestamps) unchanged...

def save(self, filename, **kwargs):
    ...policy checks (ensure_step_table / ensure_summary_table / force) stay here,
       because they call self.make_step_table()/make_summary() — that is
       orchestration, not I/O...
    cellpy_file_module.save(self.data, outfile, overwrite=overwrite)
```

`check_file_ids()` stays (it compares against *raw* files too) but its cellpy-file
half (`_check_cellpy_file`) delegates to `cellpy_file.read_fid_table()`.

---

## 4. Migration plan — small, independently-green steps

Each step ends with the full test suite green (`uv run pytest -m essential` as the
inner loop, full suite before merging; per the workspace convention use uv, not conda).

### Step 0 — Characterization tests (before touching anything)

- Round-trip: load `testdata/hdf5/..._v8.h5` → save to tmp → reload → assert frame
  equality (raw/steps/summary), meta dicts, fid list, units/limits.
- Legacy matrix: parametrized load of `_v4/_v5/_v6/_v7` files asserting shapes, key
  columns post-rename, and meta fields; `_v0` asserting the too-old failure mode.
- Selector: `load(..., selector={"max_cycle": N})` asserting summary/raw/steps are
  consistently truncated and `limit_data_points` is set.
- Failure modes: file with a missing required key → current exception type/message
  (lock in *behavior*, then improve the type in Step 6 deliberately).
- **Limits-prefix trap** *(added 2026-07-09, cross-check vs unit-handling plan)*:
  `prms._cellpyfile_raw_limit_pre_id` is the **empty string**, and the limits loop in
  `_create_infotable` (cellreader.py:2452–2455) has key/prefix usage inverted relative
  to the units loop (`new_info_table[key] = limits[h5_key]`) — it only works *because*
  the prefix is empty. Lock in with a test that limits are stored **unprefixed**;
  `format.py` must either keep the empty prefix or fix the loop deliberately, never
  give it a real prefix silently.
- Mark the fast ones `essential`.

*Deliverable: `tests/test_cellpy_file_roundtrip.py` (or extend `test_cell_readers.py`).*

### Step 1 — Create the package; move the format spec

- Add `cellpy/readers/cellpy_file/format.py` with `CellpyFileFormat` and the version
  constants (re-exported from `internal_settings` to avoid an import-cycle fight;
  `internal_settings` remains the canonical import site for now).
- Point `prms._cellpyfile_*` at the new spec (module-level aliases — zero behavior
  change).
- No call-site changes yet.

### Step 2 — Extract the stateless helpers

Move verbatim (they use no `self` state, or are already `@staticmethod`):
`_check_keys_in_cellpy_file`, `_extract_from_meta_dictionary`, `_convert2fid_list`,
`_convert2fid_table`, `_get_cellpy_file_version`, `_fix_dtype_step_table`
(the dtype-fix is storage-driven, it belongs to the writer).
Leave one-line delegating methods on `CellpyCell` temporarily so nothing else changes.

### Step 3 — Kill the side-channel: introduce `LoadSelector`/`LoadResult`

The riskiest, most valuable step — do it while the code is still inside the class:

- Rewrite `_hdf5_cycle_filter`, `_extract_summary/raw/steps_from_cellpy_file` to take
  an explicit selector/limits object and **return** limits instead of writing
  `self.limit_*`.
- `_load_hdf5_current_version` and each legacy reader thread the object through and
  return `(data, limits)`.
- `load()` assigns `self.limit_data_points` / `self.limit_loaded_cycles` from the
  result — externally identical, verified by Step 0's selector tests.

### Step 4 — Move the read path

- `read.py`: version gate (`_load_hdf5`) + v8 reader + extractors, as module functions
  taking `(store_or_path, format_spec, selector)`.
- `legacy_read.py`: v3–v7 readers + the `update_headers` rename wiring + old-meta
  unit-label conversion; register them in the version-dispatch dict.
- `CellpyCell.load()` becomes the thin wrapper of §3.3. Delete the now-dead private
  methods in the same PR (they are underscore-private; no deprecation cycle needed —
  but grep tests/ first: `test_cellpy_method_integrity.py` may assert on the method
  list and must be updated deliberately).

### Step 5 — Move the write path

- `write.py`: `_create_infotable` + `_convert2fid_table` + `_save_to_hdf5` merged into
  one `save(data, path, ...)` whose store lifecycle is a single `with` block.
- `CellpyCell.save()` keeps its policy half (§3.3).

### Step 6 — Redirect the out-of-band readers + tighten errors

- `batch_helpers.look_up_and_get()` → `cellpy_file.read_table(path, table_name,
  max_cycle=...)`; delete the hard-coded `"/CellpyData"` and the duplicated
  cycle-filter (also fixes its unclosed-store-on-non-KeyError edge: use a `with`).
- `_check_cellpy_file()` → `cellpy_file.read_fid_table()` (which owns the
  external-path temp-copy behavior).
- Introduce `CorruptCellpyFile`; keep `WrongFileVersion` semantics; narrow the
  `except AttributeError` in the wrapper to log-and-reraise-style handling that
  preserves the current public behavior for genuinely-old files (the version gate now
  catches those *before* any `AttributeError` can occur, so the blanket catch can go).

### Step 7 — Documentation, deprecations, and the upgrade CLI

- Deprecation warnings: `parent_level` argument on `load()` (docstring already says
  "Deprecating this soon!"), `prms._cellpyfile_*` aliases.
- Add `cellpy convert <old.h5> [<new.h5>]` to `cellpy/cli.py` — a thin wrapper around
  `cellpy_file.load(accept_old=True)` + `cellpy_file.save()` — the one-time upgrade
  path required by the legacy-freeze decision (§7.2).
- Update `docs/` (file-format page) to point at `format.py` as the format's single
  source of truth; note the module as the intended v2 persistence adapter and that
  cellpy v2 will read v8 files only.

Steps 1+2, 4, 5, 6 are each a reviewable PR; Step 3 can ride with 4 if the diff stays
readable, but keeping it separate makes bisection possible if selector behavior
regresses.

---

## 5. cellpy v2 / cellpy-core alignment (what this buys, later)

- **The seam contract.** cellpy-core's `metadata.io.load_archive/save_archive` are
  deliberate `NotImplementedError` stubs; `cellpy/readers/cellpy_file/` becomes the
  reference implementation the v2 app wires in. When `CellpyCell` grows
  `self.core = OldCellpyCellCore(...)` (integration plan, seam step 3), `load()`
  becomes `self.core._data = cellpy_file.load(...).data` with **no change inside the
  I/O module** — that is the test of whether this refactor drew the line correctly.
- **Format evolution becomes tractable.** A v9 format that stores meta as typed
  JSON/attrs (aligning with `cellpycore.metadata.models`, eliminating the
  pickle-protocol hack and object-dtype meta tables) is then: one new
  `CellpyFileFormat` instance, one writer function, one registry entry — instead of
  another 400-line method on `CellpyCell`.
- **Backend pluggability.** `read.py`/`write.py` orchestration vs `hdf5.py` mechanics
  is the split that lets a parquet/duckdb or BDF backend slot in under the same
  `load()/save()` API (BDF placement discussion:
  `.issueflows/04-designs-and-guides/bdf-io-placement.md`).

---

## 6. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Selector/limits side channel is subtly order-dependent (summary before raw/steps) | Step 0 selector tests lock behavior *before* Step 3 restructures it; keep extraction order identical |
| `limit_loaded_cycles` initialized from `prms.Reader.limit_loaded_cycles` (cellreader.py:298) — a config-driven global default | `LoadSelector` construction reads the same prm when no explicit selector is given; test with the prm set |
| Old-version files exercise rename paths with pandas-version-sensitive pickled objects | Keep `core.pickle_protocol(PICKLE_PROTOCOL)` wrapping all store reads/writes inside `hdf5.py`; the v0–v8 testdata files are the regression net |
| `tests/test_cellpy_method_integrity.py` may pin the `CellpyCell` method surface | Update the manifest in the same PR that deletes privates; treat any *public* method disappearance as a stop-and-reconsider signal |
| Downstream user scripts poking `cc._load_hdf5(...)` or `prms._cellpyfile_*` | Privates: accept the break (documented in HISTORY.md). Prms aliases: keep one deprecation cycle |
| `batch` link mode performance (it reads steps/summary without loading raw) | `read_table()` must keep supporting single-table reads — do not force full `load()` there |
| OtherPath/external files behave differently between `load()` (no external handling) and `_check_cellpy_file()` (temp copy) | Centralize in one boundary function; add a test with a fake-external `OtherPath` if the test rig allows |
| Merge friction with the parallel cellpy-core seam work (touches the same class header / `data` property region) | Land this refactor's Step 4 (which shrinks `load`) either clearly before or clearly after the seam PR; coordinate via issue jepegit/cellpy#377 |
| Merge friction with the unit-handling plan's Phase 1 (renames the `core` = data_structures alias and swaps `Q` imports across cellreader.py) *(added 2026-07-09)* | Land that alias/`Q` swap either before this plan's Step 2 or after Step 5 — not interleaved with the read/write-path moves |

---

## 7. Decisions taken (previously open questions)

1. **Module location: `cellpy/readers/cellpy_file/`.** Not `cellpy/io/` — a new
   top-level package would introduce a second I/O concept while `exporters/` already
   exists, and the subpackage can be moved wholesale during a v2 tree reshuffle at
   near-zero cost. Nothing in the design depends on the path.
2. **Legacy file versions freeze in v1.x; cellpy v2 reads v8 only.** The v3–v7
   readers depend on pickled pandas objects and the `update_headers` rename machinery —
   porting that to a future polars/narwhals-based `Data` is high cost for files that
   can be upgraded once by a load+save in v1. Consequences:
   - `legacy_read.py` is maintenance-only code from the moment it lands; no new
     features touch it.
   - Add a `cellpy convert <old.h5> <new.h5>` CLI command (thin wrapper around
     `load(accept_old=True)` + `save()`) in the v1.x line so users have an explicit
     one-time upgrade path before moving to v2.
   - The version-dispatch registry still keeps the door open if this decision must be
     revisited (revisit trigger: significant user base demonstrably stuck on <v8 files
     when v2 ships).
3. **`save()`'s policy checks (`ensure_step_table` / `ensure_summary_table` / `force`)
   stay on `CellpyCell` in this refactor.** They call `self.make_step_table()` /
   `make_summary()` and are therefore orchestration, not I/O. Intended end state: they
   lift to `cellpy.get`-level orchestration (or the v2 app layer) when the core seam
   lands; the §3.3 split already permits that move without touching the I/O module.

---

## 8. Estimated effort

| Step | Size | Notes |
|---|---|---|
| 0 Characterization tests | 1–2 days | Highest leverage; reuses in-tree v0–v8 files |
| 1 Format spec | 0.5 day | Mechanical |
| 2 Stateless helpers | 0.5 day | Mechanical |
| 3 Selector/limits de-stating | 1–2 days | The careful one |
| 4 Read path move | 1–2 days | Large diff, mostly cut-paste after Step 3 |
| 5 Write path move | 1 day | Includes store-lifecycle fix |
| 6 Out-of-band redirects | 1 day | batch link mode needs its own test run |
| 7 Docs/deprecations/convert CLI | 1 day | CLI is a thin wrapper + one test |

Total ≈ 7–10 focused days, spread over 5–6 PRs, each independently shippable.
