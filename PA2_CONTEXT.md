# PA2 Context — ChampSim L1D Prefetcher Assignment

## What this repo is
ChampSim trace-based microarchitectural simulator, forked for CS683 (IIT Bombay,
Autumn 2026) Programming Assignment 2. The assignment is to implement and
evaluate L1D data prefetchers against a no-prefetch baseline, using IPC and
prefetch accuracy as the metrics.

- Build: `./build_champsim.sh ${L1D_PREFETCHER} ${L2C_REPLACEMENT} ${BINARY_NAME}`
- Run: `./bin/[BINARY] -warmup_instructions N -simulation_instructions N -traces TRACE`
- Prefetcher sources live in `prefetcher/*.l1d_pref` (plain text, included by the build script).

## Current state (as of 2026-10-11)
Working tree has two modified tracked files (uncommitted, on `main`):

- `prefetcher/ip_stride.l1d_pref` — baseline IP-stride prefetcher, only a
  whitespace/blank-line diff. Effectively unchanged from upstream.
- `prefetcher/spp.l1d_pref` — custom "Stride Pattern Prefetcher" (SPP),
  ~485 lines added. Built on top of IP-stride; per-IP tracker keeps:
  - a **uniform detector** (last stride + saturating confidence counter)
  - a **non-uniform detector**: a 12-bit signature of the last 3 strides
    (4 bits/stride) indexing a 64-entry shared pattern table
    (signature → predicted next stride + confidence)
  - Decision order per demand load: uniform-confident → prefetch at constant
    stride; else pattern-confident → chain pattern predictions; else no
    prefetch. Never crosses a 4 KB page boundary.
  - An accuracy-feedback throttle adjusts prefetch degree (1–4) every 256
    filled prefetches: >60% accuracy → degree+1, <40% → degree−1.
  - Tracks "issued" (actually filled into L1D) vs "useful" (later demand-hit)
    prefetches separately from ChampSim's own counters, via an internal
    32-entry prefetch-tracking list, so reported accuracy reflects only real
    fills, not requests absorbed by MSHR/cache hits.

Three binaries have been built in `bin/` (also untracked, build artifacts):
baseline (no prefetcher), ipstride, and spp — plus `obj/` (compiled objects)
and `results/` (run logs), all untracked/gitignored-but-present.

Other stray untouched files sitting in the tree (not part of the diff, not
obviously intentional): `inc/champsim.h.bak`, `inc/champsim.h.cp`,
`inc/sedD2leEO`, `inc/kpcp.h` — worth checking whether these are editor
backups/scratch files to clean up before submission.

## Results so far (`results/ipc_summary.txt`, `results/summary_spp2.txt`)
Three traces, 25M warmup + 25M sim instructions each.

| Trace | Baseline IPC | IP-stride IPC | SPP IPC | SPP accuracy |
|---|---|---|---|---|
| trace1 | 0.232345 | 0.233452 | 0.233577 | 54.4% |
| trace2 | 0.251158 | 0.251260 | 0.251275 | 34.5% |
| trace3 | 0.382065 | 0.382862 | 0.383526 | 20.0% |

There's also an `spp2` result set (a second SPP run/variant, same binary
naming pattern `*-spp2`) with its own internal "SPP final stats" block
(prefetch requests sent, filled, useful, own-checker accuracy, uniform vs
pattern-mode prefetch counts, degree-window histogram) — IPC deltas over
ip_stride are similar in magnitude to the `spp` run, sometimes marginally
lower (e.g. trace3: 0.382927 vs 0.383526 for `spp`).

**Open question to resolve:** SPP gives only a small IPC uplift over plain
IP-stride (≤0.2%) despite much richer pattern tracking, and prefetch accuracy
is mediocre-to-poor on trace2/trace3 (20–35%). Worth investigating whether
the pattern table is being trained/looked-up correctly, whether the degree
throttle is too conservative, or whether these traces simply don't have much
non-uniform-strided behavior for SPP's pattern path to exploit.

## Why this file exists
Captures project context for continuing this work across sessions, since
none of this (assignment framing, what's implemented, where results stand)
is derivable just by reading the current diff or git log — the repo has
only 2 commits (`Initial commit`, `Update repository clone URL in README`)
and no commit yet for the SPP work itself.
