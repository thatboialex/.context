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
4. (Sonnet) Record in `baseline.md` that cloud threads cannot run `make test` (the container's proxy blocks the dependency downloads; findings 7), and ask Jason once whether to allow `github.com` in the project's cloud environment network settings. Until he does, every code phase says "host tests not run" plainly and Jason or CI runs them.
5. If Smash does not start vanilla, stop: record the crash report in baseline.md and tell the coordinator. Nothing later can work.

## Done criteria

- The protocol is complete enough that Jason can run it without asking anything.
- `baseline.md` has measured numbers for vanilla Smash, with settings and versions, including the version string the game reports (must be 13.0.5, findings 1).
- State "Closed (console-verified)" only after step 3.

## Tests and review

Review: read the protocol as Jason would; every step must be a thing he can do from the console and FTP. No code.

## Result
State: Not started
