# A1 REFLECTION

> Complete every section. CI will:
>
> 1. Verify all `## Section` headers below are present.
> 2. Verify each section has **at least 50 words**.
>
> No automatic content grading: the prose is read by a human, and the short
> prompt at the end is marked on a 0 / 0.5 / 1 scale. The numbers you quote
> in your reflection do **not** have to match canonical times exactly — HPC
> queue variance is real. Be concise, ground claims in your measurements, show your working.

## Section 1 — Schedule choice and why

Which schedule (`static` / `dynamic` / `guided` / chunk size) did you end up with, and why? Reference the cost structure of `f(x)` and what the measured timings told you. Mention at least one schedule you tried and discarded, and what the measured evidence was. Minimum 50 words.

I chose schedule(dynamic, 64) for the final version. The key feature of this kernel is that f(x) is not uniform in cost: the region x ∈ [0.3, 0.4] executes ten extra square-root iterations, so work is concentrated in a narrow contiguous part of the loop. A purely static partition can give one thread a disproportionate share of that expensive region. Dynamic scheduling avoids that by redistributing chunks as threads finish. I considered static and guided as alternative schedules, but selected dynamic, 64 because it is a natural fit for the non-uniform workload in this kernel

## Section 2 — Scaling behaviour

Looking at your `tables.csv`, where does your speedup curve depart from ideal (linear)? What does that tell you about overhead, memory bandwidth, or load balance for this kernel? Minimum 50 words.

My measured times were 3.54 s at 1 thread, 0.38 s at 16 threads, 0.20 s at 64 threads, and 0.13 s at 128 threads. That gives speedups of 9.32x, 17.70x, and 27.23x respectively. The efficiency falls from 0.58 at 16 threads to 0.21 at 128 threads. This suggests that parallel overhead and imperfect load balance both matter as thread count rises. The adaptive schedule helps, but it cannot remove all synchronization, scheduling, and reduction costs. At high thread counts, each extra thread contributes less useful work than earlier ones.

## Section 3 — Roofline position

Pick your best thread count. Using the Rome roofline constants from the day-2 slides (theoretical peak 4608 GFLOPs, HPL-achievable 2896 GFLOPs, STREAM triad 246 GB/s), what roofline fraction did you achieve against the *theoretical* and the *HPL-achievable* compute ceilings? Most non-DGEMM code (including A1) lands well below both — explain why your kernel doesn't approach DGEMM-class efficiency. If you want to argue your kernel is bandwidth-bound rather than compute-bound, justify it. Minimum 50 words.

My best measured point was 128 threads at 0.13 s. Using a conservative lower-bound count of about 0.70 GFLOP for the whole kernel, that corresponds to roughly 5.38 GFLOP/s. Relative to the Rome theoretical peak of 4608 GFLOP/s, that is about 0.12%. Relative to the HPL-achievable peak of 2896 GFLOP/s, it is about 0.19%. This kernel is far from DGEMM-class efficiency because it contains transcendental and square-root work, branching on the heavy region, and a reduction over many iterations. Those characteristics reduce vector efficiency and make the kernel much less regular than dense matrix multiplication.

## Section 4 — What you'd try next

You have two more days. What would you change about `integrate.cpp`? Pick one concrete change and predict its effect. Minimum 50 words.

If I had two more days, I would compare (dynamic, 64) against guided and a few nearby chunk sizes such as (dynamic, 32) and (dynamic, 128) on the same Rome node. The most likely benefit would be a better balance between load balancing and scheduling overhead. Smaller chunks may spread the cost more evenly, but they also increase runtime overhead. Larger chunks may reduce overhead, but risk bringing back imbalance. I would expect careful schedule tuning to improve high-thread performance more than low-thread performance.

## Reasoning question (instructor-marked, ≤100 words)

**In at most 100 words, explain why your chosen schedule is appropriate for the cost structure of this particular `f(x)`.**

(dynamic, 64) fits this loop because the cost of f(x) is not uniform: iterations in x ∈ [0.3, 0.4] do extra square-root work, so a static split can leave one thread with a heavier chunk. Dynamic scheduling lets idle threads take new chunks as soon as they finish, which improves balance. A chunk size of 64 is a compromise: it is small enough to spread the expensive region across the team, but large enough to avoid the high overhead of very tiny chunks.
