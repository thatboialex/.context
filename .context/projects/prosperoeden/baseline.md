# Baseline: vanilla Smash on the console

What Smash does with no mods, measured on Jason's console. Every later phase compares against this. Written by phase 1; the numbers come from the run in [hardware-test-protocol.md](./hardware-test-protocol.md) (Runs, phase 1), read with "Reading the logs" there.

Status: **waiting for Jason.** No row below is measured.

## Setup used

| Item | Value |
|---|---|
| ProsperoEden version / PR | waiting for Jason |
| Smash game and update version | waiting for Jason |
| Console firmware | waiting for Jason |
| Memory | 8 GiB (run C: 4 GiB) |
| Renderer, resolution, refresh rate, filter | waiting for Jason |
| Performance switches, speed limit | defaults, 100% (confirm in `stderr.log`, `Performance:` line) |
| Mods | none |

## Results

Load times are Jason's stopwatch, from Cross in the Library. Frame rate is the `heap.log` windows of the match (5 s each).

| Run | To title | To character select | Match fps min / median / mean | Worst frame (ms) | MEM at title / select / match end | GFX at match end | Notes |
|---|---|---|---|---|---|---|---|
| A: first start, 8 GiB (cold cache) | waiting for Jason | waiting for Jason | n/a | n/a | waiting for Jason | | |
| B: second start, 8 GiB (warm cache) | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | |
| C (optional): 4 GiB | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | waiting for Jason | |

Shader cache line from `stderr.log` (`Shader cache ready: N pipelines in M ms`): waiting for Jason.

## If Smash did not start

Paste the crash report's first lines and the last lines of `stderr.log` here, tell the coordinator, and stop: nothing later can work.
