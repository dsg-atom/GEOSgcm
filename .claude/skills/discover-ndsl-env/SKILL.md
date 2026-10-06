---
name: discover-ndsl-env
description: >-
  Use when standing up or re-entering the NDSL/GT4Py Python environment on NASA
  Discover for the Fork-A radiation port (running PySHiELD / pyRTE / gas-optics
  tests). Covers the per-shell re-entry block, the venv+pip (no conda) build,
  PySHiELD dependency pins, the pyRTE Fortran-compile step, the XDG_CACHE_HOME
  disk-quota fix, and GT4Py/NDSL backend selection flags. Triggers on Discover,
  NDSL, GT4Py backend, PySHiELD env, pyRTE install, venv, GTFV3_BACKEND,
  XDG_CACHE_HOME, module load GEOSpyD, pytest --backend.
---

# NDSL/GT4Py environment on Discover

How the GEOS Python side is built and run for the Fork-A radiation port. No GEOS
build is needed for port validation — PySHiELD runs standalone via pytest. Verify
paths and pins against the current repo before relying on them.

## No conda. venv + pip only.
The GEOS Python toolchain uses `python3 -m venv` + `pip`, never a conda env.
GEOSpyD is conda-origin but we build a plain venv on top of its 3.12 binary — the
"no conda" constraint is satisfied (no conda env, no `conda activate`).

## Re-entry block — run these 5 lines in EVERY fresh Discover shell
State does NOT persist across login nodes; a new shell = no module, no venv,
cwd=home. (Substitute your username / fork-a path.)
```
module load python/GEOSpyD/24.11.3-0/3.12 comp/gcc/13.2.0
export CC=gcc CXX=g++ FC=gfortran
export XDG_CACHE_HOME=/discover/nobackup/$USER/fork-a/.cache
source /discover/nobackup/$USER/fork-a/venv/bin/activate
cd /discover/nobackup/$USER/fork-a/PySHiELD
```
Confirm: `which python && python --version` → `.../fork-a/venv/bin/python`,
`Python 3.12.9`.

**XDG_CACHE_HOME is REQUIRED.** pyRTE's `get_cache_dir()` defaults to `~/.cache`
(home has a tiny quota) and the coefficient-file download hits
`Errno 122 Disk quota exceeded`. Redirect it to nobackup. (gt4py `.gt_cache` /
dace `.dacecache` land in CWD, which is already on nobackup.)

## First-time build
Template recipe mirrors `geos-gtfv3/driver/setenv/pyenv.sh`:
```
python3 -m venv $VENV_DIR
pip install -U pip wheel
# from the PySHiELD clone:
pip install -e .[test,ndsl,pyfv3]
```
- **Install the `pyfv3` extra too**, not just `[test,ndsl]` — the microphysics
  import chain needs it (`import pyshield` fails without pyfv3).
- **pyRTE-RRTMGP's pip install COMPILES Fortran** → the venv shell needs a Fortran
  compiler + netCDF-fortran + cmake loaded (the `comp/gcc` module + `FC=gfortran`
  above). This is the fiddly step.

## PySHiELD dependency pins (pyproject.toml)
- Python **>=3.12,<3.13 only**
- `ndsl @ NDSL.git@2026.08.00`, `pyfv3 @ pyFV3.git@develop`,
  `pyrte-rrtmgp @ pyRTE-RRTMGP.git@main`, `numpy>=2`, `f90nml>=1.1.0`
- Known-good resolved set: pyshield 0.2.0, ndsl 2026.8.0, pyrte-rrtmgp 0.2.1,
  pyfv3 0.2.0, gt4py 1.2.0.post22, dace 2.0.0a6, numpy 2.5.3.

## Backend selection is a flag/env var, not a rebuild
- Runtime (gtFV3-style): `GTFV3_BACKEND`, default `gt:gpu`; `is_gpu_backend()` picks
  cupy vs numpy.
- pytest: `--backend=st:numpy:cpu:IJK` (numpy CPU = correctness),
  `--backend=orch:dace:cpu:KIJ` / `gt:gpu` / `dace:gpu` (GPU timing). GPU needs
  `cupy-cuda11x` matched to the Discover CUDA.
- Other NDSL knobs: `FV3_DACEMODE=BuildAndRun`, `NDSL_LITERAL_PRECISION=32|64`,
  `GT4PY_COMPILE_OPT_LEVEL`, `NDSL_LOGLEVEL`, `GT4PY_EXTRA_COMPILE_OPT_FLAGS`.

## Running the tests on the GPU (A100) — gotchas
Correctness runs use the CPU/numpy backend on login nodes. To exercise the GPU:
- **Allocate with `--constraint=rome`** or it fails "Requested node configuration is
  not available" (gpu_a100 nodes are EPYC Rome):
  `salloc --partition=gpu_a100 --constraint=rome --ntasks=1 --gres=gpu:1 --mem-per-gpu=80G --time=1:00:00`
- **Compute nodes are OFFLINE** — pip hits `pypi.org` NameResolutionError there.
  Install cupy from a **login node** (network there; the nobackup venv is shared):
  `module load nvhpc/23.9` → `nvcc --version` → `pip install cupy-cuda12x` (CUDA 12;
  use `cupy-cuda11x` for a CUDA-11 module) → verify `python -c "import cupy"` (device
  count 0 on login is fine). Then salloc + run.
- On the A100 node, load a **CUDA module (`nvhpc/23.9`)** in addition to the CPU
  re-entry block; `python -c "import cupy; print(cupy.cuda.runtime.getDeviceCount())"`
  should print ≥1.
- The `rte_solver` GT4Py tests pick the backend from env `RTE_TEST_BACKEND` (unset =
  CPU): `RTE_TEST_BACKEND=gt:gpu pytest tests/rte_solver/test_lw_solver_gt4py.py -q`
  (`dace:gpu` may also need `FV3_DACEMODE=BuildAndRun`). Run ONE test first — the first
  GPU run triggers a slow CUDA stencil compile. Likely snag: GPU outputs are cupy
  arrays, so `assert_allclose` vs pyRTE numpy may need `cupy.asnumpy()`.
- A green GPU run proves portability/correctness on the A100, NOT model speedup
  (that needs CFFI integration + residency).

## Coefficient files
Not bundled in the pyrte_rrtmgp wheel — downloaded on first use from GitHub to the
XDG cache. GEOS L91 uses **LW_G256 + SW_G224** (+ `*_BND` cloud). Enum→file:
`LW_G128/LW_G256/SW_G112/SW_G224` = `rrtmgp-gas-{lw,sw}-g{128,256,112,224}.nc`;
`LW_BND/SW_BND` = `rrtmgp-clouds-{lw,sw}-bnd.nc`.

## What can and can't run on Discover
- Gas-optics / stencil tests run standalone (reference = pyRTE on RFMIP data). GOOD.
- The full driver test `tests/integration/test_radiation_driver.py::test_rte_rrtmgp`
  CANNOT run — it needs a hand-staged c48 SHiELD restart (`test_data/RESTART/...`,
  `eta91.nc`) that is not staged and not fetched by `get_test_data`. The end-to-end
  `step_radiation` check moves to GEOS CFFI instead.
- Known test-side ndsl drift: `tests/integration/test_radiation_driver.py` imports
  `NullComm`, which no longer exists in ndsl 2026.08.00; use `LocalComm(rank, 1, {})`
  or sidestep via `ndsl.boilerplate.get_factories_single_tile`.
