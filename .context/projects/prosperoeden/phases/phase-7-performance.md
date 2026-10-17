# Phase 7: Performance and memory with mods

**Tier:** Opus for the analysis and design; Sonnet for the settings and defaults it specifies. **Needs:** phase 5 (closed). **Wave:** 5.

## Goal

A modded Smash is no worse than a reasonable fraction of the vanilla baseline, and a heavily modded roster does not run the console out of memory.

## Read first

`baseline.md`, `findings.md` (question 8), `docs/SETTINGS.md` (Performance, Memory, Speed limit), `docs/MODS.md` (memory paragraph), `tools/analyze-perf-log.py`, `headless/performance.cpp`, `headless/cache_budget.h`.

## Scope

- Measure (protocol, phase 7 run): vanilla, Skyline only, Skyline + ARCropolis with N small ARC mods, with many. Load time, frame rate, memory.
- Decide and implement, only if the measurements justify it: a recommended memory layout note, a default for games with Skyline (the mods already ask for a game-specific `memory_layout`), Compile ahead interaction (turned off or made safe for hooked code), SD card read path (the existing fast file read patch `01-fast-file-reads.diff` may already cover it), directory listing cost.
- Record the measurements in `baseline.md` so the guide can quote real numbers.

## Out of scope

General emulator performance; other games.

## Done criteria

- Measurements for each configuration, in a table with settings and versions.
- Every change has a measurement showing why. A change without one is removed.
- No regression for vanilla Smash vs `baseline.md`.

## Tests and review

Console runs; `analyze-perf-log.py` for numbers. Opus reviewer for anything touching memory or caches.

## Result
State: Not started
