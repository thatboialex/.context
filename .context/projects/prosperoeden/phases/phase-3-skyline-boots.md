# Phase 3: Skyline loads and runs

**Tier:** Opus (design, debugging from logs); Sonnet may do the small edits Opus specifies. **Needs:** phases 0, 1, 2. **Wave:** 3.

## Goal

With Skyline installed as a ProsperoEden mod, Smash starts, Skyline initialises, and the game reaches the title screen. No ARCropolis yet; a trivial test plugin or Skyline's own log is the proof.

## Read first

`findings.md` (all of it), `baseline.md`, `headless/mods.h`, `headless/main.cpp` (how mods and the memory layout reach Eden), `tools/eden-patches/` (style and numbering), `headless/jit-*.cmake` and `jit-permissions.inc`.

## Scope

Close each Gap that findings assigns to this phase. Expected, in order of likelihood (confirm in findings):

1. `main.npdm` replacement taken from the mod's exefs.
2. `ro` service functions Skyline calls, and the SVCs for memory permission and code memory.
3. JIT invalidation after the guest writes its code (inline hooks), including the interaction with Compile ahead and the block list: disable them for a game that has a code-changing mod, or invalidate correctly.
4. Memory: Skyline's `main.npdm` asks for more; make sure `memory_layout` (4/6/8Gb) is honoured or raised.
5. Sockets used for Skyline logging: must fail without hanging when the console has no route.

## Out of scope

SD card folder (phase 2), ARCropolis behaviour (phase 5), launcher UI.

## Steps

1. Build the exact install layout the findings describe, as a mod folder under `roms/Smash/mods/Skyline/` (documented, no binaries committed).
2. Run on the console (protocol, phase 3 run). Collect logs. If it fails, read the log for the first failing service or fault; fix; repeat. Each iteration is one Opus thread or one message in a long thread; record each failure and fix in "Notes for the next session".
3. Eden changes go in numbered patches in `tools/eden-patches/`; ProsperoEden-side changes in `headless/`. Host checks for any pure logic.
4. Write what Skyline needs from the emulator in `.context/projects/prosperoeden/skyline-requirements.md`, short, for phases 5 and 6.

## Done criteria

- Console run: title screen reached with Skyline loaded, log shows Skyline's start line. (Closed only after Jason's run.)
- The same game without the mod still runs exactly as in `baseline.md` (no regression).
- Every Eden change is a patch with a one-paragraph reason in its header, reviewed by Opus.

## Tests and review

Host checks for logic; the console run for the rest. Reviewer: Opus, fresh thread, reads patches against findings. Hot-thread rule: nothing waits on the emulation thread.

## Escalate

This whole phase is Opus. If a failure needs hardware debugging beyond logs (a PS5 fault with no symbols), use `tools/symbolize-crash.py` and the crash report first; ask Jason for a second run with more detailed logging before guessing.

## Result
State: Not started
