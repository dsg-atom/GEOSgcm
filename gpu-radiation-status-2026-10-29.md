# GPU radiation port — status

**As of 29 October 2026**

Update to the [28 October status](gpu-radiation-status-2026-10-28.md). Since then
the radiation port went from correct-but-slow to measurably faster, through a
series of targeted changes. It is now 17× faster than the first GPU measurement.

## Trajectory

One radiation call, per rank (4,050 columns, 91 layers, one A100):

| Stage | Per call | Step |
|---|---:|---:|
| First GPU measurement (28 Oct) | 90 s | — |
| Gas-optics gathers moved to the GPU | 20.5 s | 4.4× |
| Cloud optics moved to the device | 13.8 s | 1.5× |
| Stop recomputing gas optics we discard | 5.3 s | 2.6× |

17× faster overall. Each change was validated to machine precision against the
pyRTE reference on the GPU before being measured.

## What changed, and why

The 28 October write-up assumed the bottleneck was a host loop over the 256
spectral slices and proposed restructuring it. A one-day prototype showed that
was wrong: the solver loop costs 0.24 s, so the loop was never the problem. The
time was in the coefficient gathers — large table lookups that were running on
the CPU.

The first two changes moved those gathers, and the cloud-optics equivalents,
from the CPU onto the GPU. The third, and largest single win, removed pure
redundancy: the code was calling the reference library to compute the full gas
optics in Fortran on every step, only to use the shape of its output and
overwrite the numbers with its own. That output container is fixed for a given
grid, so it is now built once and reused, and only the interpolation we actually
use runs each step. Nothing about the numbers changed — the same validated
answers, computed where the data already is and without the discarded work.

(One idea along the way was ruled out by measurement: collapsing the 256
spectral slices into a few large GPU kernels. It was built, correct, and made no
difference — the GPU kernels were never the bottleneck.)

## Where the 5.3 s goes now

With the redundant Fortran gone, the remaining time is the port's own code, and
no single piece dominates:

| Source | per call |
|---|---:|
| copying arrays between processor and GPU | ~2.1 s |
| GT4Py per-call bookkeeping + the minor-gas loop | ~1.7 s |
| xarray reshaping (in and out of the solver's layout) | ~0.9 s |
| the four flux solves | ~0.9 s |
| the interpolation we need | ~0.2 s |
| the actual GPU kernels | ~0.25 s |

The largest item is a round-trip: gas optics computes on the GPU, the result is
copied to host to sit in the reference library's data container, and the solver
then copies it back to the GPU.

## Remaining levers

These are bigger and harder than the changes so far, with smaller individual
payoffs. Recorded for when the work resumes:

- **Keep the optics on the GPU between gas optics and the solver (~2 s).** Stop
  routing the result through the host-side container; hand it to the solver
  on-device.
- **Batch the minor-gas loop (~1 s).** Collapse its ~1,900 small kernel calls
  into a few — harder than the others, because each minor gas spans an irregular
  set of spectral slices.
- **Speed up the xarray folding (~0.9 s)** in and out of the solver's layout.

Together these would bring a call into the low seconds, approaching native
Fortran (well under 1 second). None is a quick change.

## Status

- **Done and validated:** the port runs correctly on the GPU and is 17× faster
  than the first GPU measurement — by moving the gas-optics and cloud-optics
  gathers onto the device and building the reference output container once
  instead of recomputing (and discarding) it every step.
- **Now:** 5.3 s/call, ~5× native Fortran. The cost is the port's own host-side
  overhead, no longer anything the GPU is doing.
- **Remaining:** three bigger refactors — data residency between gas optics and
  the solver, minor-gas loop batching, and the xarray folding — to approach
  native. Each is a larger change with a smaller payoff; this is a consolidation
  point.

---

*Per-rank timing on a `gpu_a100` node (best case: one rank, one GPU). Harnesses:
`bench_step_radiation.py`, `profile_step_radiation.py` in dsg-atom/PySHiELD.
Changes on dsg-atom/PySHiELD develop: gas-optics gather-on-GPU, cloud-optics
device convert, and compute-container reuse.*
