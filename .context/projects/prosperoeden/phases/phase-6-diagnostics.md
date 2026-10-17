# Phase 6: Diagnostics

**Tier:** Haiku (Sonnet reviews). **Needs:** phase 3. **Wave:** 4 (beside phase 5). **Output:** code and docs in `thatboialex/ProsperoEden`.

## Goal

When Skyline or ARCropolis fails, ProsperoEden's logs and crash report say which step failed, so Jason can send one ZIP and a thread can diagnose it without another run.

## Read first

`skyline-requirements.md`, `headless/diagnostics.h`, `headless/crash_report.cpp`, `docs/INSTALL.md` (logs), `docs/README` Issues section.

## Scope

- At game start, log (one line each, no heavy work): Skyline files present in the applied mods, SD card folder path and whether it exists, ARC mod count, Smash version, memory layout.
- Include the SD card's `ultimate/arcropolis/logs` (or wherever findings says ARCropolis writes) tail in the crash report if it exists, size-capped. Never include other SD card contents.
- Name the first missing piece in a single log line (`Skyline: main.npdm missing from mod X`). Wording follows existing log style.
- Never log file contents or full ARC mod lists; names count only.
- A development build logs every SD/save file open at warning level (`headless/CMakeLists.txt:~1105-1143`, `EDEN_DEV_PROFILE`); a mod with thousands of files makes that log huge. Keep it out of release builds.

## Out of scope

Network log shipping; Skyline's own TCP logger (phase 3 decides whether its port stays open). Skyline's `OutputDebugString` lines already reach the log at Info in class `Debug_Emulated` (findings 4): use them rather than adding a channel.

## Done criteria

- Host check on the line formatting and the size cap (`crash_report` check pattern).
- Detailed logging off: the lines still appear (they are one-per-start) but nothing grows per frame.
- Docs: add the files to the "attach to a report" list in `README.md` Issues section.

## Tests and review

Host checks; Sonnet reviewer confirms no per-frame work and no unbounded reads.

## Result
State: Not started
