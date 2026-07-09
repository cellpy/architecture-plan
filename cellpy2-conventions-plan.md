# Plan: cellpy 2 conventions — exceptions, logging, deprecations (G6)

**Date:** 2026-07-09
**Closes:** gap-analysis item **G6**. Short by design: these are conventions every
other plan's shims and boundaries adopt, not a work package of its own.

---

## 1. Exceptions

- **One hierarchy, rooted in core.** `cellpycore.exceptions.CellpyError` is the base;
  cellpy 2's `cellpy/exceptions.py` re-exports it and derives the app-level tree:

  ```
  CellpyError (cellpycore)
  ├── NoDataFound / NullData            (cellpycore; cellpy's NullData aliases it)
  ├── WrongFileVersion                  (cellpy_file)
  ├── CorruptCellpyFile(IOError)        (cellpy_file — per the file-loading plan)
  ├── ConfigurationError                (config stack, validation failures)
  ├── UnitsError(ValueError)            (units boundary: bad label, missing scale)
  └── LoaderError                       (instrument ingestion; wraps vendor parse errors)
  ```
- **Rules:** no bare `Exception` raised anywhere (the
  `"OH MY GOD!"` raise in `_check_keys_in_cellpy_file` is the poster child); no
  blanket `except Exception` that swallows corruption (file plan §2.5 lists the
  current offenders); boundary code raises typed errors with the offending
  file/key/value named. Duplicate legacy exceptions in `cellpy/exceptions.py` get
  aliased-then-removed on the deprecation schedule below.

## 2. Logging

- **No import-time configuration.** `import cellpy` configures nothing (same
  principle as the config plan's no-import-time-I/O). Today's `log.py` +
  `logging.json` setup becomes an explicit opt-in: `cellpy.log.setup(level=...)`
  for scripts/notebooks; libraries embedding cellpy get standard
  module-level loggers (`logging.getLogger(__name__)`) and silence by default
  (`NullHandler` on the package root).
- Loggers are per-module; no `logging.debug` at module import; f-string logging
  replaced by lazy `%`-style only where profiling shows it matters (don't churn
  otherwise).
- The `Data.logger` instance attribute (data_structures.py:546) does not survive
  into cellpy 2 (frames + meta only; logging is module-level).

## 3. Warnings and deprecation policy (the shared cadence)

Every plan in this folder introduces shims; they all follow one schedule:

1. **Introduced in 2.0, removed in 2.1** unless the owning plan states otherwise
   (state it explicitly — "one release" is the default, not a law).
2. `DeprecationWarning`, **once per call site** (module-level `warnings.warn(...,
   stacklevel=2)` with a small once-registry) — never per row/loop.
3. Every shim names its replacement in the message
   (`"headers_normal.voltage_txt is deprecated; use c.schema.raw.potential"`).
4. A generated **`DEPRECATIONS.md`** table (name → replacement → introduced →
   removal) shipped in the docs; each shim registers itself so the table cannot go
   stale. Helper: `cellpy._deprecation.warn_once(name, replacement, removal="2.1")`.
5. Hard rule from the harness of plans: **private (`_`-prefixed) API breaks without
   deprecation** (file plan already applies this); public API always gets the cycle.

## 4. Validation posture

Fail loudly at boundaries, never mid-computation: config file (pydantic), unit labels
(`validate_units`), raw frames (`validate_raw_frame` strict), file keys
(`CorruptCellpyFile`). Warn-only escape hatches exist exactly twice:
`local_instrument` loading and `accept_old` file reads.

## 5. Adoption

One PR establishes the helper module + exception tree + logging change; each
subsequent plan's shims import from it. Add a checklist line to every plan's test
section: *"warnings emitted once, registered in DEPRECATIONS.md"*.
