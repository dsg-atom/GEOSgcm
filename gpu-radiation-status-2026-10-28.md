# GPU radiation port — status

**As of 28 October 2026**

The GEOS atmospheric radiation code now runs on the A100 GPU and gives correct
results. It is not yet faster. This is the measurement, why, and what a real
speedup would take.

## Bottom line

- **Works.** The full radiation calculation runs on the A100 and matches the CPU
  result to machine precision. Validated end to end.
- **Not yet worth it.** About 90 seconds per call on the GPU versus 120 on the
  CPU, and both are roughly 100× slower than the model's current Fortran
  radiation.

## The measurement

One radiation call, on the workload a single processor handles in the model:
4,050 columns, 91 layers. Same machine for both new runs; compilation excluded.

| Path | Per call | vs. Fortran |
|---|---:|---:|
| Current Fortran radiation (RRTMG) | < 1 s | reference |
| New GT4Py code, CPU backend | 120 s | ~100× slower |
| New GT4Py code, GPU backend | 90 s | ~100× slower |

GPU over CPU is 1.3×. Moving data in and out of the GPU accounts for about
0.2 s of the 90 — it is not where the time goes.

## Why it is slow

The code walks through 256 narrow slices of the spectrum one at a time, in a
loop on the ordinary processor. Each pass does a small amount of arithmetic and
a large amount of fixed setup: gathering inputs, handing a small job to the
backend, collecting the result.

That loop runs on the CPU in **both** versions. The GPU only speeds up the
arithmetic inside each pass, which is the small part, so the GPU version spends
most of its time idle, waiting on the loop that feeds it. The two backends come
out close because they share the same ~90-second loop overhead; the GPU just
removes about 30 seconds of arithmetic on top of it. That ceiling is set by the
structure of the code, not the hardware.

## What is not the problem

- **Data transfer.** Moving arrays between the processor and the GPU costs about
  0.2 of the 90 seconds. The bridge boundary is not the bottleneck.
- **GPU settings.** Tuning block sizes and similar knobs would move the needle
  slightly, not close a 100× gap.

## What a real speedup needs

Compute all 256 spectral slices together on the GPU in one pass, instead of 256
separate passes driven from the processor. That removes the per-slice setup and
gives the GPU enough work to do at once to be worth using. It is a substantial
rewrite of the gas-optics and solver code, but the target is well defined, and
the correctness work already done carries over — the rewrite only has to
reproduce known-correct answers faster.

## Where this leaves us

- **Done:** the radiation port runs on the GPU and is numerically correct.
- **Open:** whether to invest in the restructuring that turns "runs on the GPU"
  into "faster on the GPU."

---

*Figures from a single-processor timing at one rank's workload on a `gpu_a100`
node (best case: one rank, one GPU, no sharing). In the full 48-rank model,
twelve ranks share each GPU, so in-model timings would be higher. Harness:
`bench_step_radiation.py` in dsg-atom/PySHiELD.*
