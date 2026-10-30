# GPU radiation port — status

**As of 30 October 2026**

Update to the [29 October status](gpu-radiation-status-2026-10-29.md). Since then
one more change — keeping the optics on the GPU between gas optics and the
solver — took a radiation call from 5.3 s to 3.3 s. The port is now about 27×
faster than the first GPU measurement.

## Trajectory

One radiation call, per rank (4,050 columns, 91 layers, one A100):

| Stage | Per call | Step |
|---|---:|---:|
| First GPU measurement (28 Oct) | 90 s | — |
| Gas-optics gathers moved to the GPU | 20.5 s | 4.4× |
| Cloud optics moved to the device | 13.8 s | 1.5× |
| Stop recomputing gas optics we discard | 5.3 s | 2.6× |
| Keep the optics on the GPU (data residency) | 3.3 s | 1.6× |

About 27× faster overall. Each change was validated to machine precision against
the pyRTE reference on the GPU before being measured.

## What changed, and why

The 29 October profile showed the largest remaining cost was a round-trip: gas
optics computed the optics on the GPU, the result was copied to the host to sit
in the reference library's data container, and the solver then copied it back to
the GPU — with the cloud step adding two more crossings on the cloudy path. Those
are large arrays (about 0.75 GB each), and the copies were about 2 seconds of the
5.3.

The change keeps them on the GPU the whole way: gas optics → cloud combine →
solver, with no host hop. The arrays never leave the device between where they
are computed and where they are used. Nothing about the numbers changed — it is
the same arrays, moved the same way, just without the detour to the host and
back. A test runs a full radiation step both ways — resident and through the host
— and confirms every output matches bit-for-bit.

It is off by default and the model turns it on, so the earlier host path stays
intact and every existing test is unaffected.

## Where the 3.3 s goes now

The copies are gone: the measured transfer at the bridge boundary is now under a
millisecond, and the compute-only time equals the full per-call time. What is
left is the port's own compute-side work:

- the per-call bookkeeping and the minor-gas loop (~1,900 small steps),
- the array reshaping in and out of the solver's layout,
- the four flux solves,
- the interpolation we need and the actual GPU kernels (small).

## Remaining levers

Two are left, both bigger and harder than the changes so far, with smaller
payoffs. Recorded for when the work resumes:

- **Batch the minor-gas loop (~1 s).** Collapse its ~1,900 small kernel calls
  into a few — harder than the others, because each minor gas spans an irregular
  set of spectral slices.
- **Speed up the array reshaping (~0.9 s)** in and out of the solver's layout.

Together these would bring a call toward 1.5 s, close to native Fortran (well
under 1 second). Neither is a quick change, and this is a natural consolidation
point: the large, clean wins are done.

## Status

- **Done and validated:** the port runs correctly on the GPU and is about 27×
  faster than the first GPU measurement. The latest change keeps the optics on
  the GPU from gas optics through the solver, removing a ~2 s host round-trip.
- **Now:** 3.3 s/call, ~3× native Fortran. The host/device transfer is no longer
  a cost; what remains is the port's own compute-side Python work.
- **Remaining:** two bigger refactors — the minor-gas loop and the array
  reshaping — to approach native. Each is a larger change with a smaller payoff.

---

*Per-rank timing on a `gpu_a100` node (one rank, one GPU). Harnesses:
`bench_step_radiation.py`, `profile_step_radiation.py`, and the new
`test_residency_equiv.py` in dsg-atom/PySHiELD. The residency change is on
dsg-atom/PySHiELD develop (commit a8d9acf); 5.3 → 3.3 s/call measured 30 Oct.*
