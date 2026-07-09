# Plan: unit handling in cellpy 2 (built on cellpy-core)

**Date:** 2026-07-08
**Builds on:**
[data-and-cellpycell-usage-in-cellpy-utils.md](data-and-cellpycell-usage-in-cellpy-utils.md),
[hardcoded-column-headers-report.md](hardcoded-column-headers-report.md),
[legacy-cellpy-core-header-swapping.md](legacy-cellpy-core-header-swapping.md)

---

## 1. Where we stand

### cellpy-core already has (use it, don't rebuild it)

| Piece | Where | Notes |
|---|---|---|
| `CellpyUnits` unit-spec dataclass | `cellpycore/units/spec.py` | String unit labels (`charge="mAh"`, `mass="mg"`, …); promoted from legacy in issue #40 |
| Optional pint helpers (`cellpycore[units]` extra) | `cellpycore/units/converters.py` | pint lazily imported, **one memoized registry per process** (`_get_unit_registry`), stays off the compute hot path |
| `Q(...)` | same | Quantity constructor on the shared registry |
| `get_converter_to_specific` | same | gravimetric / areal / volumetric / absolute; resolves inputs from explicit kwarg → `CellMeta` → duck-typed `data`; accepts bare numbers *or* pint quantity strings (`"2 mg"`) |
| `nominal_capacity_as_absolute`, `calculate_nom_cap_abs_from_specific` | same | volumetric still `NotImplementedError` |
| `calculate_current_conversion_factor` | same | raw current unit → output unit factor for `make_core_summary` |
| `calculate_specific_conversion_factors` | same | builds the `mode -> factor` dict `add_scaled_summary_columns` consumes |
| **Factors-by-value seam** | `cell_core.py`, docs `cellpy-core-migration.md` §4 | The engines never see pint; the consumer computes floats and passes them in. This is a deliberate, documented design decision — cellpy 2 must keep it |
| `CellMeta` | `cellpycore/metadata/models.py` | `mass`, `tot_mass`, `nom_cap`, `nom_cap_specifics`, `active_electrode_area` as plain floats — implicitly in cellpy units |

### Legacy cellpy still carries (the debt cellpy 2 must retire)

1. **A second, duplicate `CellpyUnits`** in `cellpy/parameters/internal_settings.py`
   (plus a third copy re-exported by `cellpycore.legacy`). Contract test for drift is
   tracked as jepegit/cellpy#378 but the duplication itself remains.
2. **A second pint registry.** `cellpy/readers/data_structures.py` builds its own
   `_ureg`/`Q`, and — confusingly — `cellreader.py:33` imports data_structures **as
   `core`**, so every `core.Q(...)` in cellreader (e.g. the conversion factors fed into
   `make_core_summary`, cellreader.py:5894) runs on *cellpy's* registry while
   `cellpycore.units.Q` runs on *cellpy-core's*. Two registries work today only because
   nothing passes `Quantity` objects across the boundary (pint quantities from
   different registries cannot interoperate). This is a latent trap.
3. **A duplicated converter.** `CellpyCell.get_converter_to_specific`
   (cellreader.py:5169) is a hand-maintained near-copy of
   `cellpycore.units.get_converter_to_specific` (the core one is the extended port —
   same TODOs still in both docstrings).
4. **Unit methods on `CellpyCell`** with no core counterpart yet: `to_cellpy_unit`
   (:5109), `unit_scaler_from_raw` (:5149), `with_cellpy_unit` (:5080),
   `_dump_cellpy_unit` (:423).
5. **Instrument loaders declare `raw_units` as ad-hoc dicts** — the 163 unit-key
   string literals found in the hard-coded-headers review (`raw_units = {"current":
   "A", "charge": "Ah", "mass": "g", ...}` duplicated across every
   `instruments/configurations/*` module). Old v7 files stored *float scale factors*
   instead of unit strings (`convert_from_simple_unit_label_to_string_unit_label` in
   data_structures.py exists for that migration). Both core and cellpy carry the TODO
   "fix all the instrument readers (replace floats in raw_units with strings)".
6. **Plot/utils label composition** — utils reach into `c.cellpy_units.charge` etc. to
   build axis labels by hand (plotutils.py:603–836, 5971…, collectors.py:563/568) and
   use `unit_scaler_from_raw` for raw plots (plotutils.py:5307–5309).

## 2. Design principles for cellpy 2

1. **One unit spec, one registry, one converter library** — all in `cellpycore.units`.
   cellpy 2 adds *policy* (which units the session uses, where they come from), never
   *mechanism* (conversion math).
2. **Keep the factors-by-value seam.** pint remains an optional extra of the consumer
   layer; the polars engines stay pint-free.
3. **Units live at the boundaries.**
   - *Ingestion:* the instrument loader declares `raw_units` (validated `CellpyUnits`).
   - *Metadata:* stored in **cellpy units by convention** (matches `CellMeta` and the
     existing docstring contract in the spec); conversion happens once, at ingestion.
   - *Presentation:* utils ask for labels/scalers, never compose them from raw attrs.
4. **Three named unit sets, same as today, but with sharp ownership:**
   `raw_units` (per `Data`, from loader/file) → `cellpy_units` (session policy, from
   user config) → `output_units` (export/presentation). No fourth set.
5. **Strings in, floats across, quantities only inside the units module.** Public APIs
   accept bare numbers (interpreted in cellpy units) or pint quantity strings
   (`"3.579 mAh/g"`) — the pattern core's converters already implement.

## 3. Gap analysis → actions

| Capability | Legacy cellpy | cellpy-core today | cellpy 2 action |
|---|---|---|---|
| Unit spec | own `CellpyUnits` copy | `cellpycore.units.CellpyUnits` | import from core; delete copy (keep deprecated alias one release) |
| Registry / `Q` | own `_ureg` in data_structures | memoized single registry | delete cellpy's; re-export core's `Q` |
| Specific-capacity factor | `CellpyCell.get_converter_to_specific` (dup) | ✅ richer port | delegate; method becomes a 3-line wrapper |
| nom_cap → absolute | `CellpyCell.nominal_capacity_as_absolute` | ✅ | delegate |
| Current factor for summary | inline pint math (cellreader.py:5894) | ✅ `calculate_current_conversion_factor` | delegate |
| Specific converters dict | inline dict-comp (cellreader.py:5902) | ✅ `calculate_specific_conversion_factors` | delegate |
| Value → cellpy units | `to_cellpy_unit` | ❌ (has `_as_quantity` internally) | **add to core**: `convert_value(value, physical_property, from_units, to_units) -> float` |
| Raw→X scaler | `unit_scaler_from_raw` | ❌ | **add to core**: `calculate_scaler(from_unit, to_unit) -> float` (trivial: `Q(1, a).to(b).m`) |
| Meta value with unit attached | `with_cellpy_unit` | ❌ | keep in cellpy 2 as sugar over core `Q` (presentation, not conversion) |
| Loader raw-unit declaration | ad-hoc dicts, some floats historically | `CellpyUnits` + `update()` | loaders construct/validate `CellpyUnits`; add a `validate_units()` helper (pint-parsable check) |
| Unit labels for plots | hand-composed f-strings | ❌ | **new in cellpy 2**: `units_label(prop, mode=None)` (see §5) |
| Per-column unit metadata on frames | ❌ | ❌ (schema deliberately unit-free) | optional, phase 4: `df.attrs["units"]` populated at the seam |
| Energy/power/duration units | n/a (legacy frames lack the columns) | columns exist (`NATIVE_ONLY_*`), spec has `energy`, `power`, `time` | define output policy when cellpy 2 exposes native summary columns |
| Volumetric mode | partial | `NotImplementedError` in `nominal_capacity_as_absolute` | implement in core (small, well-specified) |

## 4. Phased plan

### Phase 1 — one spec, one registry (small, high value, do first)

- Replace `cellpy/parameters/internal_settings.CellpyUnits` with
  `from cellpycore.units import CellpyUnits` (deprecated alias for one release).
- Delete `_ureg` / `ureg` / `Q` / `get_ureg` from `cellpy/readers/data_structures.py`;
  re-export `cellpycore.units.Q`. The `core.Q` call sites in cellreader keep working
  because `core` is the data_structures alias — but rename that alias while at it
  (it is guaranteed to confuse every future reader now that `cellpycore` exists).
- Direct `Q` importers to fix: `instruments/neware_xlsx.py:18`, `utils/plotutils.py:5045`.
- Acceptance: one `pint.UnitRegistry` per process; jepegit/cellpy#378 contract test
  reduced to a trivial identity.

### Phase 2 — delegate the converters

- `CellpyCell.get_converter_to_specific` → thin wrapper over
  `cellpycore.units.get_converter_to_specific` (pass `data=self.data`,
  `to_units=self.cellpy_units`). Delete the duplicated body (cellreader.py:5169–5237).
- `_make_summary` factor plumbing (cellreader.py:5894–5907) → use
  `calculate_current_conversion_factor` + `calculate_specific_conversion_factors`.
- `CellpyCell.nominal_capacity_as_absolute` → delegate to core.
- Add to core (`cellpycore/units/converters.py`):
  - `convert_value(value, physical_property, from_units=None, to_units=None) -> float`
    (port of `to_cellpy_unit`, generalized: default `from_units` = raw, `to_units` =
    cellpy units; accepts number / quantity string / `(value, unit)` tuple),
  - `calculate_scaler(from_unit, to_unit) -> float` (port of `unit_scaler_from_raw`).
- `to_cellpy_unit`, `unit_scaler_from_raw` become wrappers; `with_cellpy_unit` stays
  cellpy-side (uses `Data` attributes + core `Q`).
- Acceptance: no pint arithmetic outside `cellpycore.units` and thin wrappers; parity
  tests old vs new factor values on the test fixtures.

### Phase 3 — clean the ingestion boundary

- Loaders/configurations construct `CellpyUnits(**overrides)` (or
  `CellpyUnits().update(...)`) instead of bare dicts; add
  `cellpycore.units.validate_units(units) -> CellpyUnits` that checks every label
  parses in pint and warns on unknown keys. Call it in `Data.raw_units` setter.
- Kill the float-style raw units for good: v7-file loading keeps
  `convert_from_simple_unit_label_to_string_unit_label` (quarantined in the legacy
  loader path); everything else must be strings. Resolves the shared TODO.
- Metadata convention: adopt `CellMeta` semantics — meta stored in cellpy units;
  ingestion converts once via `convert_value`. Document it on `CellMeta` and on the
  cellpy 2 setters (`set_mass`, `set_nom_cap`, …), which should accept quantity
  strings (`c.set_mass("2.3 mg")`) via `_as_quantity`.
- Acceptance: a loader with a typo'd unit fails loudly at load time, not silently at
  summary time.

### Phase 4 — presentation layer (utils/plotting)

- Add `units_label(physical_property, mode=None, *, units=None) -> str` (cellpy 2,
  e.g. `cellpy.units`): returns `"mAh"`, `"mAh/g"`, `"mAh/cm**2"`, … from the active
  spec. Replace the ~25 hand-composed f-strings in plotutils/collectors.
- Replace `unit_scaler_from_raw` call sites (plotutils.py:5307–5309) with the core
  `calculate_scaler` via the wrapper.
- Optional (recommended): at the bridge/seam, stamp `df.attrs["units"]` — a
  `column -> unit-string` dict derived from the headers mapping + unit spec — on
  summary/steps/raw frames. Plotting utils then read units per column instead of
  guessing by column *kind*. This pairs naturally with fixing the capacity-curve
  header gap from the hard-coded-headers report (§5 there).
- Acceptance: grep for `cellpy_units.` in utils returns only the label helper.

### Phase 5 — round-trip, native columns, and polish

- Persist both `raw_units` and the session `cellpy_units` in the cellpy-2 file format;
  assert round-trip in tests (today only raw_units travels with the data).
- Decide output units for the native-only summary columns (energies `Wh`, powers `W`,
  durations `sec`) before exposing them in cellpy 2 — the spec fields already exist.
- Implement volumetric mode in `nominal_capacity_as_absolute` (removes the last
  `NotImplementedError`); `CellMeta` may need a `volume` field.
- Config: `cellpy_units` initialised from user config exactly once, in one place
  (session start), not scattered `get_cellpy_units()` calls with post-hoc `update`s.
  Note: legacy prms has **no** units section — this needs the new `units:` section
  added to the config model in
  [cellpy2-configuration-and-parameters-plan.md](cellpy2-configuration-and-parameters-plan.md) §3.2.

## 5. API sketch (cellpy 2 surface)

```python
from cellpycore.units import CellpyUnits, Q, convert_value, calculate_scaler

c = cellpy.get("cell.h5")            # cellpy 2

c.cellpy_units                        # CellpyUnits (session policy, from config)
c.data.raw_units                      # CellpyUnits (validated, from loader/file)

c.set_mass("2.3 mg")                  # quantity strings accepted everywhere
c.set_mass(2.3)                       # bare number == cellpy units (mg)

c.to_cellpy_unit(0.0023, "mass")      # -> 2.3   (wrapper over convert_value)
c.unit_scaler_from_raw("mA", "current")  # wrapper over calculate_scaler

cellpy.units.label("charge", mode="gravimetric")   # -> "mAh/g"  (plots)

c.data.summary.attrs["units"]         # {"charge_capacity_gravimetric": "mAh/g", ...}
```

## 6. Testing strategy

- **Parity fixtures:** factors computed by legacy paths vs core helpers on the golden
  test cells (the same oracle style the header bridge uses).
- **Spec contract:** the single `CellpyUnits` (per #378) — becomes trivial after Phase 1.
- **Registry singleton:** test that `cellpy` and `cellpycore` quantities interoperate
  (multiply a cellpy-made `Q` with a core-made `Q` — fails today).
- **Loader validation:** every shipped instrument configuration passes
  `validate_units`; a bad label raises at load.
- **Round-trip:** save/load preserves `raw_units` + `cellpy_units`; v7 float-unit files
  still convert correctly.

## 7. Risks and open questions

1. **Two-registry trap (real today).** Any contributor who passes a `Quantity` across
   the cellpy/cellpycore boundary before Phase 1 lands gets cryptic pint errors.
   Phase 1 is cheap insurance — do it first.
2. **Frozen-by-tests seam.** The bridge's byte-for-byte oracle means unit *behaviour*
   must not change while consolidating; only the code paths may move. Parity tests are
   the guardrail.
3. **`df.attrs` durability** — pandas `attrs` propagation through operations is
   best-effort; treat frame-attached units as advisory metadata for presentation, not
   as the source of truth (the spec objects remain that).
4. **Where does `output_units` fit?** Today `get_default_output_units()` returns the
   same defaults as `cellpy_units` and is barely used. Decide in Phase 5: either give
   it a real job (export formatting) or drop it to avoid a third half-alive namespace.
5. **Temperature** — `"C"` is a risky pint label (Coulomb vs Celsius; pint wants
   `degC`). Worth auditing while implementing `validate_units`.
6. **Per-cell unit overrides** — the schema/units-by-value design supports different
   raw units per cell cleanly; make sure batch utils in cellpy 2 never assume shared
   raw units across a batch (they implicitly do today in a few label helpers).

---

## 8. Cross-check against the companion plans (2026-07-09)

Checked against the other cellpy-2 planning documents in this folder. No hard
contradictions; the coordination points below were folded into the respective
documents (marked *added/amended 2026-07-09* there).

**Confirmed agreements**

- *Units are not metadata*: [cellpy2-metadata-handling-plan.md](cellpy2-metadata-handling-plan.md)
  §2 principle 5 and its open question 2 recommend exactly this plan's §2.3 convention
  (plain floats in cellpy units on `CellMeta`; pint-free core; quantity-string
  coercion only at the API boundary). This plan effectively answers that open question.
- *Factors-by-value seam*: restated independently by the config plan (requirement 5),
  the metadata plan (boundary rule), and this plan (principle 2).
- *Death of the prefix hack*: this plan Phase 5 ↔ metadata plan G3/Step 4 ↔
  polars-index report §2.11; the file-loading plan correctly keeps the prefixes alive
  only as frozen v8 format spec.
- *v7 float-unit conversion* is quarantined in exactly one place: the legacy read path
  (this plan Phase 3 = file-loading plan Step 4 `legacy_read.py`).

**Coordination points (now recorded in the other plans)**

1. `units:` section added to the config model
   ([config plan](cellpy2-configuration-and-parameters-plan.md) §3.2) — legacy prms
   has no units section, so Phase 5's "from config" needed a home; `override(units=…)`
   gives scoped unit overrides for free.
2. `ScienceDefaults` values declared to be in cellpy units by convention (config plan
   §3.2 bullet) so the metadata resolver's lowest layer can't disagree silently.
3. `CellMeta.volume` added to the metadata plan's Step-1 decision list (needed by this
   plan's Phase 5 volumetric mode).
4. Metadata persistence sketch amended to explicit `"raw_units"` + `"cellpy_units"`
   keys (metadata plan Step 4) — matches this plan's Phase 5 round-trip requirement.
5. Per-test `raw_units` registered as metadata-plan open question 6 (merged tests from
   different instruments may carry different raw units; recommendation there: reject
   mixed-unit merges in v2.0, defer per-test units until a real use case).
6. Sequencing: this plan's Phase 1 (`core` alias rename + `Q` swap in cellreader.py)
   must land before the file-loading refactor's Step 2 or after its Step 5 (risk rows
   added in both plans). The loader-contract changes (this plan Phase 3 + metadata
   plan Step 3) should ship as one PR series — same files, same shape of change.

**SPEED-30 convergence (added 2026-07-09, from the gap analysis F1)**

cellpy-core's `column-headers-review.md` recommends a future unit+dtype-carrying,
versioned header object (SPEED-30 / `SuperDuperCols` prototype). That is the *end
state* for per-column units; this plan's Phase-4 `df.attrs["units"]` device and the
`units_label()` helper are the interim. Design both so their lookup can be re-pointed
at SPEED-30 headers later (i.e. resolve units through one function, not scattered
`attrs` reads), and do not build a second competing per-column-unit mechanism.
Post-header-swap, `units_label()` vocabulary follows the native names
(`potential`, not `voltage`) with legacy aliases through the deprecation shim.

**Verified facts**

- The cellpy file stores only `raw_units` (strings enforced at save) + `raw_limits`;
  session `cellpy_units` does not travel. The `cellpy_units_` prefixing at
  cellreader.py:3415 is the Excel export, not the cellpy file.
- `prms._cellpyfile_raw_limit_pre_id == ""`, and the limits loop in
  `_create_infotable` (cellreader.py:2452–2455) is inverted relative to the units loop
  — it works only because the prefix is empty. Characterization-test note added to the
  file-loading plan's Step 0.
