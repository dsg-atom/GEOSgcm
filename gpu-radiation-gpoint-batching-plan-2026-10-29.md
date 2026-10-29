# GPU radiation — speedup plan

**Plan of 29 October 2026, revised the same day after the prototype below.**

Follow-on to the [status of 28 October](gpu-radiation-status-2026-10-28.md): the
GPU radiation port is correct but ~90 s/call. This plan locates where the time
goes and how to remove it. The first hypothesis was wrong; a cheap prototype
caught that before any large rewrite, and the revised plan is narrower.

## Update: the prototype changed the diagnosis

The first idea was that a host-side Python loop over the 256 spectral slices
(g-points) was the bottleneck, and that folding the g-point axis into the GPU's
parallel domain would remove it. We built that as a prototype on the longwave
solver — the cleanest case — and measured it before committing.

Result, at the model's per-rank size (4,050 columns, 91 layers, 256 g-points):

- The fold is numerically exact — it reproduces the loop to 1 part in 10^15.
- It gave **no speedup: 1.0×**.
- The longwave solver is only **0.24 s/call**, all 256 launches included.

So the 256-g-point loop is not the bottleneck. The four solves together are about
1 second of the 90. The other ~89 seconds is **gas optics**, and since 256
GPU-loop iterations cost only ~0.24 s, it is not the kernel launches either. It
is the **host-side coefficient gathers** in gas optics — large numpy
fancy-indexing operations (`col_mix`, the eight-corner `fmajor`, `fminor`,
`jeta`) run on the CPU, 256 times plus the minor-gas loop, which the solver does
not do.

The prototype cost less than its 1–2 day budget and ruled out a 3–6 week refactor
built on the wrong premise. That is the result, not a detour.

## Progress (29 October, later the same day)

The gather-on-GPU lever below was built and measured, and a cheap follow-on with
it:

- Gas-optics gathers moved to the GPU: **90 s → 20.5 s** per call (4.4×).
- Cloud optics moved to the device (same pattern): **20.5 s → 13.8 s** (1.5×).

Both validated to machine precision against pyRTE on the GPU. See the
[29 October status](gpu-radiation-status-2026-10-29.md) for the trajectory and
the profile.

**The batching below was then built for the major-gas step and measured — and it
made no difference.** It was numerically correct but gave no speedup, which means
the spectral-slice launches were never the bottleneck. A full profile showed the
remaining ~13 s is host overhead, dominated by the reference library recomputing
the gas optics we discard (~5–6 s), xarray reshaping (~4 s), and host/GPU copies
(~2 s); the GPU kernels total under half a second. So the next lever is **not**
the batching in the section below — it is removing that recomputation (run the
reference compute once for its output container, reuse it, keep only the one
interpolation we use). The batching section is retained as the record of an
approach that was tried and ruled out by measurement; it is not the plan going
forward.

## Revised lever: run the gas-optics gathers on the GPU

The fix is narrower than folding the domain, and it does not touch the stencils.
Keep the existing per-g-point loop; move the gathers onto the GPU. Convert the
coefficient tables and index arrays — `col_mix`, `fmajor`, `fminor`, `jeta`, and
the `si/ei/li/iflav` indices — to device arrays so that
`col_mix[si, ei, li, iflav]` runs on the GPU instead of the CPU.

Why it should work: the solver prototype shows 256 GPU-loop iterations cost
~0.24 s. Gas optics runs the same number of iterations; its extra cost is the
host gathers. Move those on-device and gas optics should fall from ~89 s toward
the ~1 s the solver already reaches.

## Scope and the one wrinkle

- Convert the gather operands in `_shared_interp`, `_tau_absorption`,
  `_tau_rayleigh`, and `_planck` to the backend array module — cupy on GPU, numpy
  on CPU, via the helper already in the code.
- The wrinkle is mixed host/device indexing. The g-point flavor map is a host
  table indexed per cell by the tropopause mask, so the flavor index has to be
  computed on-device too; otherwise the gather mixes a host table with a device
  index and fails.
- Re-validate against the pyRTE reference throughout; the existing comparison
  tests are the guardrail.

## Cost

Days, not weeks. It is a targeted change to the gas-optics gather setup, measured
with the existing `bench_step_radiation` harness. If the measurement confirms the
drop, the same conversion extends to the minor-gas loop and cloud optics. The
memory-safe multi-rank layout — the thing that crashed GPU nodes on 28 October —
is still required before any in-model GPU run, and is tracked separately.

## What was ruled out

Folding g-points into the parallel domain (`lw_solver_batched.py`): validated and
numerically exact, but no speedup, because the loop was never the cost. Kept as a
validated layout exercise; not extended.

---

*Grounded in the current code in dsg-atom/PySHiELD: `lw_solver.py` /
`gas_optics.py` (stencils), `gas_optics_gt4py.py` (the gather loops),
`lw_solver_batched.py` (the prototype). Timing basis: `bench_step_radiation.py`
and `bench_lw_solver_batched.py`.*
