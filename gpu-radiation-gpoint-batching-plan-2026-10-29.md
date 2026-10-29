# GPU radiation — plan to batch all g-points in one pass

**As of 29 October 2026**

Follow-on to the [status of 28 October](gpu-radiation-status-2026-10-28.md). That
timing showed the GPU radiation port is correct but ~90 s/call, bottlenecked by a
host-side loop over the 256 spectral slices (g-points). This is the plan to remove
that loop, and an assessment of what it costs.

## The change

Today every radiation stencil runs on one tile of 4,050 columns × 91 layers, and
the 256 g-points are a Python `for` loop *around* that stencil: 256 iterations,
each a host-side coefficient gather, a tiny GPU kernel launch, and a copy-back.
The g-points are physically independent — they meet only at the final broadband
sum — so that loop is overhead that happens to run on the CPU.

The change: make the g-point axis part of the GPU's parallel work instead of a
host loop. One stencil call then covers all (column × g-point) pairs at once, and
a single sum over the g-point axis gives the broadband flux. The GPU goes from
256 jobs of ~368K elements each to one job of ~94M elements — enough to saturate
an A100.

## Two pieces, very different difficulty

**The solvers — straightforward.** The longwave no-scattering and shortwave
two-stream solvers are per-column vertical sweeps (forward/backward over the
layers). The stencil math does not change. Build the compute tile as
`ncol × ngpt` columns instead of `ncol`, run the existing sweep once, and replace
the in-stencil broadband accumulation with one reduction over the g-point axis
afterward. The vertical recurrence is what the stencil framework is good at, so
this is mostly a layout and host-driver change.

**Gas optics — the real work.** `_tau_absorption` (major + minor gases),
`_tau_rayleigh`, and `_planck` don't just compute; they *gather* coefficient-table
entries whose indices depend on the g-point, the tropopause, and the
interpolation indices. Those lookups are currently baked into the stencil through
a per-call scalar g-point index. To do all g-points at once, each lookup must be
hoisted out and done as one vectorized device gather producing
`(column, layer, g-point)` arrays, leaving the stencil as pure arithmetic over the
big tile. Because that arithmetic is then pointwise, gas optics can be written with
the numpy/cupy array helper already in the code rather than the stencil framework
— same CPU/GPU portability, simpler. The awkward part is the minor-gas loop: each
minor gas contributes to a *variable* g-point range with masks, so it vectorizes
as a masked scatter-add rather than a clean reshape.

## The catch: memory, so chunk

The folded tile is ~1M columns. A single `(column, layer, g-point)` array is
~755 MB, and the major-gas interpolation alone needs eight corner-gathers (~6 GB)
before the other terms. All 256 at once would be tens of GB — fine on a dedicated
40 GB A100, impossible at the 12-ranks-per-GPU layout that crashed nodes on
28 October.

So the real design is **g-points in chunks** — e.g. 32 at a time, 8 chunks. That
cuts 256 launches to 8 (removing ~99% of the host overhead), keeps peak memory
near 1 GB, and makes the chunk size a knob tuned to available GPU memory. It is
also what makes the eventual in-model, multi-rank run memory-safe.

## What it should buy

The ~90 s per call is almost entirely the host loop and its gathers. Remove it and
what remains is one vectorized gather (memory-bandwidth bound, a few milliseconds
on the A100's ~2 TB/s) plus pointwise arithmetic over 94M points (sub-second). A
realistic target is **well under a second per call**, which would put GPU
radiation ahead of the CPU backend and plausibly near the native Fortran, instead
of ~100× behind it.

This is an estimate from hardware numbers, not a measurement — which is why the
first step is a prototype, not the full rewrite.

## Risks

- **Memory** — handled by chunking; the main thing to get right.
- **Gather correctness** — the flavor/tropopause/g-point indexing is fiddly; the
  existing pyRTE comparison tests catch errors, so re-validation is the guardrail
  throughout.
- **Minor-gas irregularity** — the one piece that does not vectorize cleanly.
- **Compile time** — should get *easier*, not harder: a larger domain of the same
  stencil, not a 256× unrolled one.
- **Framework fit** — gas optics likely leaves the stencil framework for array
  expressions; the solvers stay. A reasonable split, not a rewrite of everything.

## Cost, and how to spend it

| Step | Effort | Purpose |
|---|---|---|
| **Prototype: longwave solver only** — fold g-point into the domain, reduce, measure time and peak memory vs the 256-loop | ~1–2 days | Decisive: proves the speedup and the memory/compile behavior on the cleanest case before any gas-optics work |
| Shortwave solver (same pattern) | ~days | |
| Gas optics: major + planck + rayleigh (hoist gathers, vectorize, chunk) | ~1–2 weeks | The bulk of the work |
| Minor-gas masked scatter-add | ~1 week | The trickiest piece |
| Cloud-optics combine step | ~days | |
| Re-validation + chunk/memory tuning + memory-safe multi-rank layout | threaded throughout | |

Roughly **3–6 weeks** of focused work by someone who knows this code, gated on a
**1–2 day prototype**.

## Recommendation

Build the longwave-solver prototype first. It is small, it is the cleanest case,
and its measured per-call time and peak memory — folded-domain solver vs the
current 256-loop — decide whether the full gas-optics gather rewrite is worth
starting. If it hits near its target, the rest is mechanical though laborious; if
it does not, we have spent two days instead of six weeks.

---

*Grounded in the current code: `lw_solver.py` / `sw_solver.py` (solver stencils),
`gas_optics.py` / `gas_optics_gt4py.py` (gather + g-point loop), `cloud_optics.py`
in dsg-atom/PySHiELD. Timing basis: `bench_step_radiation.py`.*
