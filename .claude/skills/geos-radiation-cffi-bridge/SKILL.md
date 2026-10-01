---
name: geos-radiation-cffi-bridge
description: >-
  Use when wiring GT4Py/NDSL radiation (PySHiELD) into GEOS Fortran via a CFFI
  embedding bridge, mirroring the merged gtFV3 bridge. Covers the Fortran->C->Python
  CFFI embedding pattern, the init/run/finalize triple, R8 (float64) array
  marshalling and its 8-byte copy-back bug, the whole-column-tile residency rule,
  and the default-off build + runtime guards. Triggers on CFFI, ffi.embedding_api,
  geos_gtfv3_interface, FV_StateMod, GEOSradiation_GridComp, bind(c), ISO_C_BINDING,
  embedding Python into GEOS, GEOS radiation integration.
---

# GEOS ↔ GT4Py radiation CFFI bridge

How to embed the GT4Py radiation driver into GEOS Fortran. The template is the
**already-merged, in-tree gtFV3 bridge** in GEOS-ESM/FVdycoreCubed_GridComp
(`develop`, PR #412), read-only local copy at
`../gpu-analysis/FVdycoreCubed_GridComp/geos-gtfv3/`. Mirror its accepted shape.
Verify file:line references against current code — they drift.

## Accepted precedent (lead with this when pitching)
- gtFV3's CFFI bridge is **in GEOS's own upstream tree, fully wired, but off by
  default**. Files: `geos_gtfv3_interface.{f90,c,py}`, `f_py_conversion.py`,
  `geos_gtfv3.py`, `cuda_profiler.py`, `driver/`.
- Wired into `FV_StateMod.F90`: `geos_gtfv3_interface_f_init` (~L1208), `..._f`
  (~L2011), `..._f_finalize` (~L2441), all under `#ifdef RUN_GTFV3`.
- **Off by default twice:** build guard `option(BUILD_GEOS_GTFV3_INTERFACE "..." OFF)`
  (the Python `.so` isn't compiled unless flipped on); runtime guard `#ifdef RUN_GTFV3`
  + MAPL resource `RUN_GTFV3:` default `0`.
- Do NOT overstate it: say "merged as an opt-in capability," not "the production
  GPU path." It is experimental/opt-in.

## Bridge layers (top to bottom)
1. **Fortran `bind(c)` interface** (`*_interface.f90`): ISO_C_BINDING declarations
   for init/run/finalize; MPI communicator passed as a C int.
2. **C shim** (`*_interface.c`): `MPI_Comm_f2c` to turn the Fortran comm into a C
   comm; declares the `@ffi.def_extern` externs the Python side implements.
3. **CFFI embedding** (`*_interface.py`): `ffi.embedding_api(header)` +
   `ffi.embedding_init_code(source)` + `ffi.set_source(...)` + `ffi.compile()` →
   `libgeos_*_interface_py.so` that embeds CPython. The `@ffi.def_extern()` functions
   are the init/run/finalize bodies.
4. **Pure-Python singleton driver** (`geos_*.py`): holds the GT4Py driver instance
   across calls (built once at init, reused each run).

## init / run / finalize triple
- **init:** build the driver, factories, grid indexing, allocate device state once.
  **FPE-trap workaround:** importing numpy/driver can raise SIGFPE; GEOS wraps init
  with `ieee_set_halting_mode(..., .false.)` and restores after. Replicate this.
- **run:** marshal GEOS state in, call `step_radiation`, marshal fluxes out.
- **finalize:** free device state, teardown.

## Array marshalling (f_py_conversion.py) — R8 is the catch
- gtFV3 marshals **float32**; `cp.asarray` (H2D) on entry, `cp.asnumpy` (D2H) on exit.
- **RRTMGP is float64 (R8).** The bridge must marshal float64 end to end.
- **Known template bug:** `f_py_conversion.py` hardcodes `4 * numpy_array.size`
  (sizeof float32) in the `memmove` copy-back (~line 325). For R8 this must be
  **8 bytes** or the copy-back under-copies and corrupts fluxes. Fix the byte count
  (and any other sizeof-float32 assumptions) when adapting for radiation.

## Residency rule (the performance crux)
- GEOS feeds radiation in blocks of ≤4 columns (`RRTMGP_{SW,LW}_BLOCKSIZE`, default 4)
  under an OMP parallel-do — the tiny-launch antipattern (~3,000 tiny launches/rank/
  rad-step). On a PCIe-bound host that collapses any kernel speedup.
- **The bridge must hand the WHOLE column tile at once:** `ncol = nx*ny`.
  `GasOpticsGT4Py` asserts `ncol == nx*ny`. One H2D in, one D2H out per hourly
  radiation refresh — not per block.
- Bigger win later: read state already on-device from gtFV3/pyMoist (shared cupy
  device memory) so radiation never round-trips at all.

## Build wiring (CMake)
- `add_custom_command` runs `mpirun -np 1 python geos_*_interface.py` to compile the
  embedding `.so`; an INTERFACE library exposes it to the Fortran target.
- Guard with `option(BUILD_GEOS_RADIATION_GT4PY_INTERFACE "..." OFF)` so a stock
  build is byte-for-byte unchanged. GEOS build's Python is whatever CMake
  `Python3_EXECUTABLE` finds → the venv must be on PATH at configure time.

## Where the GEOS-side call-out goes
- Add it to a **dsg-atom fork of GEOSradiation_GridComp**, guarded by its own build
  option (OFF) + runtime MAPL resource flag (default 0), so stock GEOS is unchanged.
- The superproject `components.yaml` repoint to the fork is a **GEOSgcm-level change
  → flag it to the user first**; do not assume a local-only patch is acceptable.
