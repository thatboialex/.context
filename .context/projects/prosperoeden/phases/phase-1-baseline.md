# Phase 1: Hardware baseline and test protocol

**Tier:** Sonnet. **Needs:** nothing. **Wave:** 1 (runs beside phase 0). **Output:** a short `baseline.md` boot check and a finished `hardware-test-protocol.md`; no code.

## Goal

Know that vanilla Smash runs on Jason's console. If it does not, nothing later can work, and we find out now instead of after the Skyline work. Numbers (load time, frame rate, memory) are not needed for that; phase 7 takes them if it wants a comparison.

## Read first

`hardware-test-protocol.md`.

## Steps

1. (Sonnet) Write the minimal run in the protocol: set up vanilla Smash 13.0.5, start it, reach character select, play one match, quit, send the logs only if it failed. Done.
2. (Sonnet) Create `baseline.md` with the boot check table, "waiting for Jason".
3. (Jason) Do the run, and say whether it passed.
4. If Smash does not start vanilla, stop: record the crash report in baseline.md and tell the coordinator.

## Done criteria

- The protocol is complete enough that Jason can run it without asking anything.
- `baseline.md` says whether vanilla Smash reaches character select and plays a match.
- State "Closed (console-verified)" only after step 3.

## Tests and review

Review: read the protocol as Jason would; every step must be a thing he can do from the console and FTP. No code.

## Result
State: Draft PR
PR: https://github.com/thatboialex/.context/pull/17
Changed: `hardware-test-protocol.md` (minimal Setup, Every run, phase 1 run; timings and log reading moved to "Measuring" for phase 7); `baseline.md` (boot check, waiting for Jason); this file; the plan's and README's status; phase 7.
Unverified: everything about the console; step 3 is Jason's.
Notes for the next session: The frame-rate line (`EDEN_VULKAN_FRAME fps= worst_ms=`, every 5 s) is in `heap.log` of release builds, but `tools/analyze-perf-log.py` needs `EDEN_DEV_*` lines that only `make dev` prints; the protocol has an awk one-liner. Load times are not logged. Logs rotate to `.prev.log` when ProsperoEden next opens: copy them first. Set State to Closed (console-verified) when Jason reports the boot check.