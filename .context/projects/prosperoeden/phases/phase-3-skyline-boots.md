# Phase 3: Skyline loads and runs

**Tier:** Opus (design, debugging from logs); Sonnet may do the small edits Opus specifies. **Needs:** phases 0, 1, 2. **Wave:** 3.

## Goal

With Skyline installed as a ProsperoEden mod, Smash starts, Skyline initialises, and the game reaches the title screen. No ARCropolis yet; a trivial test plugin or Skyline's own log is the proof.

## Read first

`findings.md` (all of it), `baseline.md`, `headless/mods.h`, `headless/main.cpp` (how mods and the memory layout reach Eden), `tools/eden-patches/` (style and numbering), `headless/jit-*.cmake` and `jit-permissions.inc`.

## Scope

Findings 3 to 6 say Eden already has what Skyline calls, so this phase starts as **verification on the console** and writes code only for what the first logs show missing. In order:

1. `main.npdm` and `subsdk9` taken from the mod's exefs: check the log lines named in findings 3. If the mod's `main.npdm` is ignored, find out why before changing anything.
2. `ro`, `MapProcessMemory`, own-process handle, `fsp-srv` SD: present at the pin; the log shows the first one that fails.
3. JIT invalidation after inline hooks, with Compile ahead **off** first (the run with it on is in phase 7): Skyline's hooks must take effect. Findings 6 lists what to look at if they do not.
4. Memory: Skyline's own cost; `memory_layout` is already per game (4/6/8Gb).
5. Sockets: Skyline binds TCP port 6969 on any address (findings 4). It must fail without hanging when the console has no route. Decide with Jason whether to leave the port open, and tell phase 6.
6. Load address: Skyline needs pages before the entry point (Eden's comment, findings 9). Fix the wrong comment in `headless/load_address.h` (seed 0 is `0x80000000`, not what ARCropolis checks for) when this phase touches the area.

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
