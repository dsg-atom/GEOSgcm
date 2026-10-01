---
name: geos-gcm-run-setup
description: >-
  Use when running gcm_setup to create a GEOS AGCM experiment for GPU-porting
  profiling or a validation run, or when deciding resolution / vertical levels /
  physics-stack options. Gives the ordered interactive prompt sequence with
  recommended answers and the reason L91 (not L72) is required to exercise the
  GPU-ported code paths (RRTMGP radiation, GFDL_1M, GF2020). Triggers on gcm_setup,
  GEOSgcm experiment setup, AMIP run, c180, c360, L91, LM 91, vertical resolution,
  RRTMGP vs RRTMG, GFDL_1M, profiling run config.
---

# gcm_setup for a GPU-porting profiling / validation run

Ordered interactive prompts from `gcm_setup` (GEOSgcm, NCCS/Discover site) with
answers chosen for an atmosphere-only (AMIP) run aligned to the GPU-ported code
paths. Verify prompt order and option labels against the current `gcm_setup` — it
changes between versions.

## The one answer that matters most: LM = 91
`gcm_setup` makes the vertical level count flip the whole physics stack:
- **L72 = v11 stack:** RRTMG radiation, BACM_1M. **Wrong for us** — RRTMG has no
  `accel/` kernels and is not the scheme being ported.
- **L91+ = v12 stack:** **RRTMGP** radiation + **GFDL_1M** microphysics + GF2020
  convection + Land v12. This auto-aligns the run with every GPU-ported path
  (pyMoist = GFDL_1M + GF2020; radiation = RRTMGP). gtFV3 dynamics is
  resolution-independent.

## Ordered prompts + answers
1. **Experiment ID** → free label (e.g. `gpu_c180_L91`)
2. **Experiment Description** → free text
3. **CLONE an old experiment?** → ENTER (No)
4. **Atmospheric Horizontal Resolution** → `c180` to bring up / iterate; `c360` for
   the real assessment
5. **Vertical Resolution LM** → **`91`** ⭐ (see above)
6. **Microphysics** → ENTER (defaults to GFDL_1M at L91)
7. **IOSERVER?** → `FALSE` (fewer moving parts)
8. **Processor Type** (`mil` Milan / `cas` Cascade) → ENTER (`mil`). Profile on the
   CPU partition for clean per-component timers — NOT the a100 nodes; gcm_setup
   offers no GPU node type.
9. **COUPLED Ocean/Sea-Ice?** → ENTER (No) → AMIP prescribed-SST
10. **Data_Ocean Horizontal Resolution** → ENTER (`CS` cubed-sphere OSTIA) — pairs
    with the cube grid, no SST regrid
11. **Land Surface Boundary Conditions** → ENTER (`v12` default at L91)
12. **Land Surface Model** → ENTER (`1` Catchment) — option 2 adds biogeochem cost
13. **GOCART Actual or Climatological Aerosols?** → `C` — climatological, lighter
    data staging
14. **GOCART Emission Files** → ENTER (`AMIP`)
15. **HEARTBEAT_DT** → ENTER (resolution-dependent default)
16. **HISTORY template** → ENTER; trim output later if needed
17. **EXPDIR / HOMDIR paths** → accept defaults
18. **group_list / SBU billing group** → user's NCCS project group

A conditional "Data Atmosphere?" prompt exists on the OGCM=FALSE branch but may not
appear.

## Profiling notes
- MAPL has built-in hierarchical timers (`CAP.rc: MAPL_ENABLE_TIMERS: YES`, default
  on). The global profiler prints a Name / #cycles / Inclusive% / Exclusive% tree on
  rank 0 at finalize. Rank by Exclusive% for leaf kernels, Inclusive% at a subsystem
  root for total reclaimable time.
- This build does NOT echo "EGRESS" to stdout — clean-finish signal is the profiler
  timer tree printing + auto-resubmit of the next segment.
- Radiation refresh is hourly (`SOLAR_DT/IRRAD_DT/SATSIM_DT=3600` → 24 solves/day),
  so radiation's measured share is real, not inflated by per-step calls.
- Compare profiles across resolutions by **cells/rank**, not rank count: match
  cells/rank (≈1,620 in the baseline runs) or decomposition differences will dilute
  component shares for non-physics reasons.

## Measured component ranking (CPU baseline, context for where porting pays)
c180 L91 AMIP: RADIATION ~33% > MOIST ~18% > DYN ~16.5%. c360: RADIATION 26.7% >
DYN 23.1% (already gtFV3) > MOIST 17.5%. Radiation is the #1 un-ported target in
both. Design rule on this PCIe-bound host: maximize on-device residency, port a
contiguous majority of the timestep as one resident block, don't offload isolated
kernels. Needs BCs/restarts staged on Discover (standard BOUNDARY_DIR); report
missing-file failures.
