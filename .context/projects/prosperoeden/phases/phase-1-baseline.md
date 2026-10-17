# Phase 1: Hardware baseline and test protocol

**Tier:** Sonnet. **Needs:** nothing. **Wave:** 1 (runs beside phase 0). **Output:** `baseline.md` and a finished `hardware-test-protocol.md`; no code.

## Goal

Know that vanilla Smash runs on Jason's console, how well, and how to measure later changes against it. Without this no later phase can say whether it made things worse.

## Read first

`hardware-test-protocol.md`, `headless/hud.h` and `tools/analyze-perf-log.py` (what the performance log holds), `docs/SETTINGS.md` (Performance, Memory, Speed limit), `docs/INSTALL.md`.

## Steps

1. (Sonnet) Write the exact commands and settings for the baseline run in the protocol's Setup, including how to start the performance log and which file `tools/analyze-perf-log.py` reads. Add what to record: load time to title and to character select, frame rate in a match (HUD: Touchpad + R1), memory in use, settings used.
2. (Sonnet) Create `baseline.md` with an empty results table for the fields above and the vanilla run's row marked "waiting for Jason".
3. (Jason) Do the run; post the logs. A Haiku or Sonnet thread fills `baseline.md` from them with `analyze-perf-log.py` output.
4. If Smash does not start vanilla, stop: record the crash report in baseline.md and tell the coordinator. Nothing later can work.

## Done criteria

- The protocol is complete enough that Jason can run it without asking anything.
- `baseline.md` has measured numbers for vanilla Smash, with settings and versions.
- State "Closed (console-verified)" only after step 3.

## Tests and review

Review: read the protocol as Jason would; every step must be a thing he can do from the console and FTP. No code.

## Result
State: Draft PR
PR: see the `.context` draft PR for branch `claude/project-thread-g9eikk`
Changed: `hardware-test-protocol.md` (Setup, Every run, Reading the logs, phase 1 run); `baseline.md` (new, empty results table, every row "waiting for Jason"); this file; the plan's and README's status.
Unverified: everything about the console. Steps 1 and 2 are done from source and docs only; step 3 (the run) and step 4 are Jason's.
Notes for the next session: The frame-rate line (`EDEN_VULKAN_FRAME fps= worst_ms=`, every 5 s) is in `heap.log` of release builds, but `tools/analyze-perf-log.py` needs `EDEN_DEV_*` lines that only `make dev` prints, so it prints nothing useful for a release baseline; the protocol has an awk one-liner instead. Load times are not logged, so they are Jason's stopwatch. Logs rotate to `.prev.log` the next time ProsperoEden opens: copy them before reopening. Fill `baseline.md` from Jason's ZIP, then set State to Closed (console-verified).
