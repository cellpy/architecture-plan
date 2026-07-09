# Plan: redesign of the ica utils (dQ/dV **and** dV/dQ, a specced frame, a pure core)

**Date:** 2026-07-09
**Closes:** the wave-3 design gap in the
[utils migration plan](cellpy2-utils-migration-plan.md) §2 ("`utils/ica.py`:
port; dq/dv on top of `curves`; scipy stays app-side; **spec the ICA output
frame while at it**"). Fourth doc in the utils redesign series, after
[batch](cellpy2-batch-redesign-plan.md),
[plotting](cellpy2-plotting-redesign-plan.md) and
[collectors](cellpy2-collectors-redesign-plan.md).
**Foundation:** [loader plan](cellpy2-loader-port-and-extraction-plan.md)
§2.3 (`cellpycore.curves`), [conventions plan](cellpy2-conventions-plan.md).

---

## 1. Review: what ica.py is

`utils/ica.py` is 1 124 lines and — unlike batch/plotting/collectors — its
core is in decent shape: the `Converter` five-stage pipeline
(set → inspect → pre-process → increment → post-process,
`ica.py:27–417`) implements a defensible dQ/dV recipe (interpolate V(q),
optional Savitzky-Golay pre-smoothing, invert to q(V), diff, gaussian
post-smoothing, area normalization), and **`tests/test_ica.py` has 19
tests** — the best characterization net of any util reviewed so far. The
problems are at the edges: API sprawl, hidden state, silent failure, and
rot.

### 1.1 Four entry points × three ways to pass the same 20 options

| Entry | Input | Output |
|---|---|---|
| `dqdv_cycle` (`ica.py:433`) | one cycle frame | tuple of numpy arrays |
| `dqdv_cycles` (`ica.py:555`) | multi-cycle frame | long frame (`cycle, voltage, dq`) — or `(keys, list-of-frames)` if `not_merged` |
| `dqdv_np` (`ica.py:659`) | raw arrays | tuple of numpy arrays |
| `dqdv` (`ica.py:787`) | CellpyCell | long frame, **or** a `(charge, discharge)` pair of wide MultiIndex frames if `split=True` |

The ~20 `Converter` parameters can arrive (a) as `Converter(**kwargs)`
constructor args, (b) tunneled through `**kwargs` of every entry point, or
(c) as `dqdv_np`'s 17 explicit parameters that are then copied onto the
converter through ten consecutive `if x is not None` blocks
(`ica.py:733–776`). The same 25-line "Keyword Args" block is copy-pasted
into three docstrings (`ica.py:449–469, 574–599, 808–836`) — already
drifting (`dqdv_np` spells the smoothing flag `diff_smoothing`; everywhere
else it is `smoothing`).

### 1.2 Two data paths that can disagree

The combined path (`_dqdv_combinded_frame` — typo and all, `ica.py:899`)
gets its curves from `cell.get_cap(method="forth-and-forth", ...)`. The
split path (`_dqdv_split_frames`, `ica.py:937`) instead calls
`collect_capacity_curves` from `cellpy.readers.data_structures` (analysis
code living in the *readers* package) with its own
`trim_taper_steps`/`steps_to_skip` options that the combined path does not
have. Same function name, different extraction semantics depending on
`split=` — and the split path builds its output as a list of DataFrames
carrying data in an ad-hoc `.name` attribute (`ica.py:869–896`), pasted
into wide MultiIndex frames. The `tidy` parameter means long-format in
`dqdv` but defaults differ per function (`dqdv(tidy=True)`,
`_dqdv_split_frames(tidy=False)`).

### 1.3 Silent failure and hidden state

- `dqdv_cycle` wraps each half-cycle in `except Exception` and, on *any*
  error, substitutes **empty arrays** (`ica.py:510–514, 534–538`) — a
  cycle can silently lose its charge branch; downstream plots simply show
  nothing. The ValueError-retry (disable post-smoothing and try again) is
  copy-pasted for both half-cycles (`ica.py:495–503, 523–531`).
- `Converter.inspect_data` **mutates option state**: it overwrites
  `normalizing_factor` from the data (`ica.py:218–224`), so a reused
  converter carries the previous cycle's normalization into the next.
- Direction semantics are implicit: the first half-cycle is
  `direction == -1`, the last is `+1` (`ica.py:485–486`) — charge vs
  discharge is left for the reader to infer, and the output column is
  named `dq` although it holds dQ/dV (units depend on the `normalize`
  option, documented nowhere).

### 1.4 Rot

- `from scipy.ndimage.filters import gaussian_filter1d` (`ica.py:14`) —
  deprecated namespace, **already warning on scipy 1.18** in the dev
  environment; will break on a future scipy.
- `increment_method="hist"` is half-implemented ("has not been thoroughly
  tested yet", `ica.py:329–349`) with TODO markers for a contributor
  ("assigned to Asbjoern", `ica.py:24`) that date back years. The
  nom-cap-normalization TODO appears twice (`ica.py:612–616, 844–848`).
- ~105 lines of `_check_*` dev functions plus an `if __name__ == "__main__"`
  block (`ica.py:1020–1124`) with hard-coded testdata paths.
- Duplicated Savitzky-Golay window arithmetic in `pre_process_data` and
  `increment_data` (`ica.py:250–260, 298–308`).

### 1.5 The missing sibling: dV/dQ (DVA)

cellpy computes only dQ/dV. Differential voltage analysis (dV/dQ vs
capacity) — the standard technique for electrode balancing and
degradation-mode analysis, particularly for full cells — does not exist
anywhere in cellpy or cellpycore. Some instrument readers *ingest* a
`dv_dq` column when the cycler exports one (`arbin_sql_csv.py:35`,
`maccor_txt_one.py:24`, ...), but there is no way to compute it from the
curves. Ironically, the existing `Converter` pipeline computes the needed
intermediate — the smoothed V(q) representation *before* the inversion to
q(V) — and discards it; dV/dQ is one `np.gradient` away from data the code
already has in hand.

### 1.6 Consumers

`collectors.ica_collector` (→ `dqdv`), `batch_tools/batch_helpers.py:391`
(`_extract_dqdv` → `dqdv_np`, feeding batch's `export_ica` CSVs),
`easyplot` (three call sites; module already scheduled for removal), the
plotting plan's future `ica_plot` family, and the tests. The cellpy-core-dev
validation notebook `03_capacity_curves.py` exercises `dqdv_cycles`
end-to-end and doubles as a visual check.

---

## 2. Design principles

1. **Two public verbs, one machinery.** `dqdv()` and `dvdq()` accept a
   cell, a curves frame, or two arrays; everything else becomes a
   deprecated alias. Input polymorphism replaces the four-function menu,
   and both derivatives ride the same extraction/smoothing pipeline —
   dV/dQ (DVA) is new capability, not new machinery (§1.5).
2. **One options object.** `IcaOptions` dataclass is the single source of
   truth for the recipe parameters — one definition, one docstring, shared
   by function, class shim and collectors.
3. **A specced output frame.** Long format, always the same columns:
   `cycle, direction, voltage, dqdv` for ICA and
   `cycle, direction, capacity, dvdq` for DVA (each carries the other
   coordinate as an extra column so both curves can be cross-plotted).
   Wide format is an explicit converter, not an entry-point mode. This is
   the "spec the ICA output frame" item from the utils plan, and the
   contract the plotting plan's `ica_plot`/`dva_plot` families and the
   collectors plan's `collect_ica` build on.
4. **Pure core, thin state.** The math becomes a stateless function
   `transform_half_cycle(voltage, capacity, options) -> (v, dqdv, meta)`;
   options are never mutated by data inspection (derived quantities like
   the effective normalization factor live in the returned meta).
5. **One data path.** All curve extraction via the curves contract
   (`cellpycore.curves` when it lands; `c.get_cap` until then). The
   `collect_capacity_curves` side door from the readers package closes.
6. **Failures are visible.** Per-half-cycle errors are collected and
   reported (warning summarizing which cycles/directions failed);
   `strict=True` raises. Empty output is never a silent substitute.
7. **Scipy stays here.** The ICA math remains cellpy-side (cellpycore is
   scipy-free); if the binning method ever materializes as a polars-native
   algorithm it can move to core later without changing this API.

---

## 3. Proposed shape

A single module — `cellpy/ica.py` (shim kept at `cellpy.utils.ica`); ica is
too small to justify a package:

```python
@dataclass(frozen=True)
class IcaOptions:
    # interpolation
    voltage_resolution: float | None = None
    capacity_resolution: float | None = None
    max_points: int | None = None
    interpolation_method: str = "linear"
    # smoothing
    pre_smoothing: bool = False
    diff_smoothing: bool = False
    post_smoothing: bool = True
    savgol_window_divisor: int = 50
    savgol_order: int = 3
    voltage_fwhm: float = 0.01
    gaussian: GaussianOptions = ...
    # normalization
    normalize: Literal["area", "nom_cap", False] = "area"   # today's True == "area"
    normalizing_factor: float | None = None
    normalizing_roof: float | None = None
    # differentiation
    increment_method: Literal["diff"] = "diff"   # "hist" pending §6 Q2

def dqdv(source, cycles=None, direction="both", options=None,
         strict=False, **overrides) -> pd.DataFrame:
    """source: CellpyCell | tidy curves frame | (voltage, capacity) arrays.
    Returns the specced long frame: cycle, direction, voltage, dqdv.
    Frame .attrs carry units, options used, and per-cycle failure notes."""

def dvdq(source, cycles=None, direction="both", options=None,
         strict=False, **overrides) -> pd.DataFrame:
    """Differential voltage analysis (DVA) — same sources and options.
    Returns: cycle, direction, capacity, dvdq (voltage carried along).
    NEW capability (§1.5): computed as dV/dq on the smoothed V(q)
    representation the pipeline already builds — no q(V) inversion step,
    so it is numerically *simpler* than dQ/dV."""

def to_wide(ica_frame) -> pd.DataFrame          # explicit wide conversion
def transform_half_cycle(voltage, capacity, options,
                         derivative="dqdv") -> HalfCycleResult
                                                 # the pure core (public: it IS
                                                 #   dqdv_np's use case);
                                                 #   derivative="dqdv" | "dvdq"
```

- `direction="charge" | "discharge" | "both"` replaces `split=True` (the
  old split output = `to_wide(frame[frame.direction == ...])`).
- The `dq` column is renamed `dqdv` with the old name kept as a duplicate
  column behind a deprecation flag for one release (collectors and
  plotting migrate in the same wave).
- `Converter`, `dqdv_cycle`, `dqdv_cycles`, `dqdv_np` remain as thin
  deprecated wrappers over the new core for one minor release — the 19
  existing tests keep passing against the shims until they are ported.
- The retry-without-post-smoothing fallback becomes one code path in
  `transform_half_cycle` (logged into result meta, not copy-pasted).
- `taper trimming` (`trim_taper_steps`/`steps_to_skip`) becomes an option
  of the curve-extraction call, available to *both* directions and the
  combined path — closing the §1.2 asymmetry.
- The scipy import is fixed (`scipy.ndimage`), the savgol window helper is
  extracted, dev `_check_*` functions move to tests/notebooks.
- Normalization semantics differ per derivative: `normalize="area"` is an
  ICA concept; for `dvdq` the default is `normalize=False` (DVA is
  conventionally plotted raw, with peak *positions* on the capacity axis
  carrying the information). Options validation rejects meaningless
  combinations instead of silently applying them.

Estimated size: **~500–600 lines** (from 1 124), most of it the ported
math; dV/dQ itself adds only ~30 lines to the core.

---

## 4. Migration plan

Wave 3 in the utils plan sequencing (after plotting Phase 2, alongside
`ocv_rlx`). Small enough to land as one arc:

**Phase 0 — extend the net (½ day).** The 19 tests are the base; add
golden *numeric* snapshots of `dqdv` output for two cells from
`cellpy-core-dev/data` (values, not shapes) and a charge/discharge-split
case, so the §1.2 unification is measurable. Add a failing-cycle test
capturing today's silent-empty behavior (to be flipped in Phase 1).

**Phase 1 — pure core + options (1–1.5 days).** `IcaOptions`,
`transform_half_cycle`, scipy-import fix, state-mutation fix, error
collection. Old entry points re-implemented on the core; golden tests must
match (except the documented silent-empty → warned-failure change).

**Phase 2 — the new `dqdv` + `dvdq` + spec (1–1.5 days).** Input
polymorphism, specced frames with `.attrs`, `to_wide`, direction handling
on the single data path; `dvdq` lands here as the pure core's second
derivative mode, validated against instrument-exported `dv_dq` raw columns
where available (Arbin SQL test data) as a sanity reference. Old functions
become deprecation shims. Docs: one options table replaces the three
copy-pasted kwargs blocks.

**Phase 3 — consumers (½–1 day).** `collect_ica` (collectors plan §3.2)
and the plotting plan's `prepare/ica.py` consume the specced frame;
`batch_helpers._extract_dqdv` re-pointed or retired with batch's export
rework; tests ported off the shims.

**Phase 4 — removals (2.x cadence).** Shims (`Converter`, `dqdv_cycle`,
`dqdv_cycles`, `dqdv_np`, `dq` column alias) out in 2.1;
`collect_capacity_curves` in the readers package deprecated in favor of
the curves contract.

Total: **~3.5–4.5 days**, consistent with the utils plan's wave-3 budget
(the extra half day is the new dV/dQ capability, not overhead).

---

## 5. Risks

| Risk | Mitigation |
|---|---|
| Numeric drift in the port (interpolation/smoothing order is subtle) | Golden numeric snapshots from Phase 0; port is a move, not a rewrite — the math functions transplant nearly verbatim |
| Silent-empty → visible-failure changes notebook behavior | It only *adds* a warning; output frames gain rows only where errors were previously eaten. Release-notes item with example |
| `normalize=True` default surprises (output units depend on it) | Kept as default (`"area"`), but now documented in the options table and recorded in frame `.attrs` |
| `hist`/binning method has an intended future | Decision forced in §6 rather than carrying dead branches; the pure-core seam makes adding a second `increment_method` trivial later |
| scipy pin drift on py3.14+ | The import fix removes the known time bomb; scipy usage is otherwise stable API (`interp1d`, `savgol_filter`, `gaussian_filter1d`, `simpson`) |

---

## 6. Open questions for the maintainer

1. **Column rename `dq` → `dqdv`** in the specced frame — worth the
   one-release dual-column shim, or keep `dq` forever?
2. **The `hist`/binning increment method** (the old "assigned to Asbjoern"
   TODO): finish it as `increment_method="bin"` with tests, or delete the
   branch? (This plan works either way; the seam is designed for it.)
3. **Normalization to nominal capacity** (the twice-repeated TODO):
   promote to `normalize="nom_cap"` in this rework (small: factor plumbing
   already exists via `normalizing_roof`), or defer?
4. `dqdv`/`dvdq` for full cells (the module-header TODO): in scope for the
   port, or a separate feature after cellpy 2.0? Note that `dvdq` makes
   this more pressing — electrode balancing from full-cell DVA (fitting
   half-cell reference curves) is the classic application, and would be a
   natural later extension on top of the specced frame.
5. Should `transform_half_cycle` be public API (recommended: it is the
   honest replacement for `dqdv_np` power users), or private?
