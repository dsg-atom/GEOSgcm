# GPU radiation port — status

**As of 29 October 2026**

Update to the [28 October status](gpu-radiation-status-2026-10-28.md). Since then
the radiation port went from correct-but-slow to measurably faster, through two
targeted changes. It is now 6.5× faster than the first GPU measurement, with one
large lever left.

## Trajectory

One radiation call, per rank (4,050 columns, 91 layers, one A100):

| Stage | Per call | Step |
|---|---:|---:|
| First GPU measurement (28 Oct) | 90 s | — |
| Gas-optics gathers moved to the GPU | 20.5 s | 4.4× |
| Cloud optics moved to the device | 13.8 s | 1.5× |

6.5× faster overall. Both changes were validated to machine precision against the
pyRTE reference on the GPU before being measured.

## What changed, and why

The 28 October write-up assumed the bottleneck was a host loop over the 256
spectral slices and proposed restructuring it. A one-day prototype showed that
was wrong: the solver loop costs 0.24 s, so the loop was never the problem. The
time was in the coefficient gathers — large table lookups that were running on
the CPU.

The two changes since then moved those gathers, and the cloud-optics equivalents,
from the CPU onto the GPU. The arrays now live on the device, so the lookups and
the per-slice work happen there instead of on the processor. Nothing about the
numbers changed — the same validated answers, computed where the data already is.

## Where the 13.8 s goes now

A profile of one call:

| Component | per call |
|---|---:|
| Gas optics | ~12.2 s |
| The four flux solves | ~0.9 s |
| Writing results back + marshalling | ~0.7 s |

Gas optics is now ~88% of the time — and it is not computing. The GPU kernels are
microseconds each. The cost is the overhead of launching thousands of small
kernels, a separate batch for each of the 256 spectral slices. Moving data to and
from the GPU is negligible (~0.04 s).

## The remaining lever

Compute all 256 spectral slices in a few large kernels instead of 256 small ones.
That collapses the launch overhead that now dominates. It is the batching
described in the plan, and the profile confirms gas optics is where it pays — the
solver, when measured, did not need it.

This is the larger piece of work, roughly 1–2 weeks: the gas-optics coefficient
gathers have to be reworked to produce all slices at once, and the minor-gas
contributions use irregular slice ranges that do not batch cleanly. It is the
only remaining path to close most of the gap to the native Fortran radiation
(well under 1 second per call).

## Status

- **Done and validated:** the port runs correctly on the GPU and is 6.5× faster
  than the first GPU measurement, through moving the gas-optics and cloud-optics
  gathers onto the device.
- **Remaining:** the gas-optics slice batching, to remove the launch overhead
  that is now ~88% of the per-call cost. Still ~14× slower than native until that
  lands.

---

*Per-rank timing on a `gpu_a100` node (best case: one rank, one GPU). Harnesses:
`bench_step_radiation.py`, `profile_step_radiation.py` in dsg-atom/PySHiELD.
Changes: gas-optics gather-on-GPU and cloud-optics device convert on
dsg-atom/PySHiELD develop.*
