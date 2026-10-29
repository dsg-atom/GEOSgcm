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

Gas optics is now ~88% of the time — and, as a deeper profile showed, it is
barely computing. The GPU kernels total under half a second. The time is host
overhead, and most of it is work we throw away (see below).

## What the remaining cost actually is

The plan expected the remaining cost to be the overhead of launching a separate
GPU batch for each of the 256 spectral slices, and proposed collapsing those into
a few large kernels. That was built for the major-gas step and measured: it was
numerically correct and made **no difference** to the time. So the launches were
never the bottleneck.

A full profile then showed where the ~13 s goes:

| Source | per call |
|---|---:|
| pyRTE's own Fortran gas optics (run, then discarded) | ~5–6 s |
| xarray reshaping (folding data in and out of the solver's layout) | ~4 s |
| copying arrays between processor and GPU | ~2 s |
| the actual GPU kernels | < 0.5 s |

The single largest piece is pure redundancy: the code calls the reference library
to compute the gas optics — the full Fortran calculation — only to use the shape
of its output, then overwrites the numbers with its own. The reference
interpolation (which we do need) runs twice as a result.

## The remaining lever

Stop recomputing what we discard. Run the reference computation once to get the
output container, reuse that container on later steps, and keep only the one
interpolation call we actually use. That removes most of the ~5–6 s, with the
xarray reshaping and the host/GPU copies as smaller follow-ons. It is host-side
work, not GPU work — the GPU core is already fast. This is the path toward the
native Fortran speed (well under 1 second per call).

## Status

- **Done and validated:** the port runs correctly on the GPU and is 6.5× faster
  than the first GPU measurement, by moving the gas-optics and cloud-optics
  gathers onto the device.
- **Learned:** the remaining cost is host-side, dominated by the reference
  library recomputing gas optics we discard — not GPU work, and not the
  spectral-slice launches the plan assumed (that change was built, measured, and
  made no difference).
- **Remaining:** remove that recomputation (and the smaller reshaping and copy
  overheads). Still ~14× slower than native until it lands.

---

*Per-rank timing on a `gpu_a100` node (best case: one rank, one GPU). Harnesses:
`bench_step_radiation.py`, `profile_step_radiation.py` in dsg-atom/PySHiELD.
Changes: gas-optics gather-on-GPU and cloud-optics device convert on
dsg-atom/PySHiELD develop.*
