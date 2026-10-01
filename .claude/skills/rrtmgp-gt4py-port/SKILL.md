---
name: rrtmgp-gt4py-port
description: >-
  Use when porting a GEOS/RRTMGP radiation kernel (gas optics, RTE solver, cloud
  optics, aerosol optics, Planck sources) to GT4Py/NDSL stencils in the dsg-atom
  PySHiELD fork, or when writing a translate/validation test for such a port.
  Covers the Fork-A method: port a Fortran kernel to GT4Py, validate it bit-for-bit
  against compiled pyRTE as the oracle. Triggers on RRTMGP, GT4Py, GasOpticsGT4Py,
  gas optics, tau, Planck, RTE solver, lw_solver_noscat, GlobalTable gather,
  pyRTE, PySHiELD radiation.
---

# RRTMGP → GT4Py porting (Fork-A method)

This is the validated method for moving a piece of GEOS RRTMGP radiation onto the
GPU as GT4Py/NDSL stencils. Everything here was learned by actually doing it
(gas-optics gather core is complete and class-level validated). Verify file:line
references against current code before relying on them — they drift.

## Strategic frame (do not re-litigate)
- **One GPU framework: all GT4Py.** No OpenACC in the runtime path. Mixing OpenACC
  kernels with GT4Py stencils reinstates the two-GPU-worlds seam. The banked
  OpenACC LW solver is a **bit-exact reference oracle only**, never runtime code.
- **Fidelity is the pitch to GEOS:** "provably reproduces your computations" via
  bit-for-bit validation. So every ported kernel must match an oracle to tolerance.
- Repo to edit is the **dsg-atom/PySHiELD** fork (branch `develop`). Analysis
  clones under `../gpu-analysis/` are READ-ONLY. Never write to GEOS-ESM upstream
  or the GEOSgcm superproject.

## The oracle: compiled pyRTE
pyRTE-RRTMGP (installed in the venv) is Python bindings that COMPILE the reference
RTE+RRTMGP Fortran. It is the golden reference. Two levels:
- Module-level thin f2py wrappers: `import pyrte_rrtmgp.rrtmgp as R` →
  `R.interpolation(...)`, `R.compute_tau_absorption(...)`, `R.compute_planck_source(...)`.
- High-level xarray driver on `BaseGasOptics`: `go.interpolate(atm, gas_mapping)`,
  `go.tau_absorption(atm, interp)`, `go.compute_planck`, `go.compute_sources`,
  `go.compute`. Reuse these to build the reference — don't reconstruct bookkeeping.
- **All pyRTE indices are 1-BASED (Fortran). GT4Py stencils are 0-BASED → subtract 1
  when feeding reference indices in.**

## Non-negotiable process lessons (each cost real debugging time)
1. **Read the installed pyRTE source before writing a test. Never guess the API.**
   Dump the actual `tau_absorption`/`compute_planck`/`compute_sources` source and the
   dtypes/dims on Discover first. Wrong guesses already made: `pyrte_rrtmgp.data_types`
   and `rrtmgp_gas_optics` modules do NOT exist; there is no `__version__`.
2. **The float32-promotion trap (NEP50).** `play`/`tlay`/temperatures are stored
   float32. `Python-float × float32-array` stays float32 under NumPy weak promotion,
   diverging ~1 float32-ulp from the Fortran's float64 and producing a ~1e-7 gap.
   FIX: `.astype(np.float64)` on the float32 fields before any factor like
   `0.01*play/tlay`. This bit the minor-tau density factor and the Planck temp interp.
3. **Isolate contributions by ZEROING, not subtraction.** Minor tau is O(1e-8)
   inside major tau O(1–40); subtracting loses ~7 digits. To get a major-only
   reference, zero the kminor tables (`xr.zeros_like`); minor-only, zero kmajor.
4. **Stop guessing after one failed fix.** If a port is off, read the reference
   source and dump dtypes immediately rather than trying another blind tweak.

## GT4Py capabilities that are PROVEN usable (and the ones that aren't)
- **Multi-axis value-indexed table gather** via `GlobalTable`: declare
  `MyTab = GlobalTable[(Float, (n0,n1,n2))]` (tuple = dtype, data_dims; no spatial
  axes). Read-only inside the stencil with `tab.A[i,j,k]` ("A global indexation");
  WRITING to `.A[...]` is forbidden. Pass the table as a plain NumPy ndarray kwarg
  (not wrapped in a Quantity), shape+dtype matching the alias.
- **Integer index arithmetic inside `.A[...]`** works: `jtp = jtemp+1` computed
  in-stencil and used as a gather index is accepted. No need to pass +1 neighbors
  as extra IntFields.
- **No `for` loop in a stencil body.** This gt4py.cartesian frontend has
  `visit_While` but NO `visit_For` — `for g in range(...)` raises
  `GTScriptSymbolError: Unknown symbol 'range'`. It also unrolls data-dim accesses
  at compile time, so a runtime data-dim WRITE index is unsupported.
  → **Iterate g-points with a Python (host) loop that drives one stencil compiled
  once**, each call writing a slice into the output's "gpt" data axis. Expect N
  stencil launches per contribution (e.g. 256 for LW) — launch overhead, not a
  correctness problem; data stays on-device.
- Index fields are `IntField` (`ndsl.dsl.typing`). `floor`, `log` import from
  `ndsl.dsl.gt4py`. Scalar externals typed as plain floats work.

## The GasOpticsGT4Py pattern (the template to mirror)
- Class wraps a pyRTE `GasOptics`, loads+pads all coeff tables ONCE in `__init__`,
  builds the stencils, precomputes minor index / gpoint_flavor bookkeeping.
- **Overwrite-first container:** `.compute(...)` calls `self._pyrte.compute(...)` to
  get the fully-structured output Dataset and set accessors, re-runs `interpolate`,
  then OVERWRITES the same-named data vars in place with the GT4Py stencil results.
  pyRTE `interpolate` + `.rte.solve` are the only pieces deliberately kept on host —
  the solver is the next port target.
- `_tile`/`_write` fold the atmosphere's non-core "column" dims into the stencil tile
  (nx,ny) and back, using the SAME dim order both ways (per-cell stencils are
  pointwise). Driver case: single flattened `column` axis = nx*ny row-major.
  `assert ncol == nx*ny`.
- **Pad-and-reuse tables across LW/SW** rather than writing new stencils: SW kmajor
  `(14,60,9,224)` → transpose to `(14,9,60,224)` → zero-pad gpt 224→256 to fit the
  LW `KMajor` alias; pad tail is never read.

## Validation workflow
1. Stand up the env (see the `discover-ndsl-env` skill) — needs no GEOS build.
2. Build the reference from pyRTE's high-level driver on real RFMIP data (and the
   GEOS-L91 coeff files: LW_G256, SW_G224). Save to `.nc` on Discover nobackup.
3. Write the stencil(s). Drive host-side loops (flavor, g-point) around a
   compile-once stencil.
4. Test: compare mine vs pyRTE to the tolerances the gas-optics port used — tau
   rtol 1e-9, ssa 1e-8, g exact, Planck 1e-10 / jacobian 1e-9, toa 1e-12.
   Mirror `tests/gas_optics/test_gas_optics_gt4py.py`.
5. Commit to dsg-atom/PySHiELD `develop`. The gas-optics work landed across commits
   e7b75b3 → 34085f0; follow that cadence (one validated stage per commit).

## Port order (dependency order; each a checkpoint)
gas optics (DONE) → **RTE solver** (lw_solver_noscat + SW two-stream; next, shared
by both builds, currently still CPU pyRTE) → cloud optics → aerosol optics →
broadband flux reduce + heating rates. Cloud/aerosol are where GEOS and GFS physics
diverge — those must re-express GEOS's own formulas, not reuse PySHiELD's GFS ones.
