# Plan: Skyline and ARCropolis on ProsperoEden

Status: Proposed, 2026-10-17; phase 0 findings in, 2026-10-17. Requested by Jason. Code: none yet. How threads run it: [agent-workflow.md](./agent-workflow.md).

## Goal

Start Super Smash Bros. Ultimate from ProsperoEden with Skyline and ARCropolis loaded, so custom characters (ARC mods) work, with the player supplying every file. Setup is a short guide. A failure leaves a log that says which step failed.

Non-goals: online play with mods, bundling or downloading Skyline/ARCropolis/mods ([ADR 001](./decisions/001-user-supplied-skyline.md)), other Skyline games (kept possible, not tested), HDR/other big mod packs beyond a custom-character smoke test.

## Background (checked in phase 0, see [findings.md](./findings.md))

Citations and the remaining unverified points are in `findings.md`; the answers here are the short form.

1. Skyline is installed to a game as `exefs/subsdk9` and a replaced `exefs/main.npdm` (Smash's, with every SVC and service allowed). Plugins are `.nro` files in `romfs/skyline/plugins/`. Confirmed (findings 2).
2. ARCropolis (`libarcropolis.nro`) reads the SD card: ARC mods in `sd:/ultimate/mods/<name>/`, its config per user under `sd:/ultimate/arcropolis/config/`, and it hooks the game's file loading. The hashes file is **optional** (log lines only), not required (findings 2).
3. Smash must be at exactly **13.0.5**, the only version ARCropolis 4.1.1 runs on (findings 1). The update goes beside the game in `roms/<Game>/`.
4. Skyline patches game code at run time (inline hooks), loads NROs with the `ro` service and opens a TCP log port. Eden at the pin has all of it: `main.npdm` layering, `ro`, SD card, `MapProcessMemory`, own-process handle, code-cache invalidation (findings 3 to 6). Only the console can show that it works on the PS5.
5. ProsperoEden recognises `main`, `main.npdm`, `subsdk0..9` in a mod's `exefs/` (`headless/mods.h` `Kinds`), and Eden layers them: the install should already load (findings 3). The SD card also exists already, at `/data/prosperoeden/user/sdmc`, undocumented (findings 5). What it lacks: documentation of the SD card, setup guidance, status in the launcher, Skyline-specific diagnostics, memory defaults for a heavily modded Smash.
6. **Compile ahead** and the JIT caches: the JIT invalidates on `ic ivau` across all cores and Compile ahead drops stale blocks by design (findings 6). Phase 3 and 7 test this on the console.
7. **New:** ARCropolis tells an emulator from a console by the main module's address (`0x8004000` or `0x8504000`); Eden and ProsperoEden load at `0x80000000`+, so it takes the console branch and runs `ninput::init()` (findings 9). Phase 5 finds out if that matters.

## Phases

Each phase is one unit a fresh session can run from its file plus the code. Tier names the agent that fits: **Haiku** (mechanical, docs, small edits), **Sonnet** (normal implementation and review), **Opus** (deep emulator reasoning, design, hard debugging from logs). Details and done criteria are in each phase file.

| # | Phase | Tier | Needs | Wave |
|---|---|---|---|---|
| 0 | [Findings: what Eden and ProsperoEden lack](./phases/phase-0-findings.md) | Sonnet | nothing | 1 |
| 1 | [Hardware baseline and test protocol](./phases/phase-1-baseline.md) | Sonnet | nothing | 1 |
| 2 | [SD card folder](./phases/phase-2-sd-card.md) | Sonnet | 0 | 2 |
| 3 | [Skyline loads and runs](./phases/phase-3-skyline-boots.md) | Opus | 0, 1, 2 | 3 |
| 4 | [Launcher: status and switch](./phases/phase-4-launcher.md) | Sonnet | 2 | 3 |
| 5 | [ARCropolis loads a custom character](./phases/phase-5-arcropolis.md) | Opus (Sonnet for fixes) | 3 | 4 |
| 6 | [Diagnostics: logs that name the step](./phases/phase-6-diagnostics.md) | Haiku | 3 | 4 |
| 7 | [Performance and memory with mods](./phases/phase-7-performance.md) | Opus analysis, Sonnet fixes | 5 | 5 |
| 8 | [Guide, changelog, qualification](./phases/phase-8-guide.md) | Haiku | 5, 6, 7 | 6 |

```
0 ─┬─> 2 ─┬─> 3 ─┬─> 5 ──> 7 ─┐
1 ─┘      │      └─> 6 ───────┼─> 8
          └─> 4 ──────────────┘
```

Waves are the parallel groups: the coordinator starts every phase of a wave as its own thread once the previous wave is merged. Phases 3 and 4 touch different files (Eden patches and `headless/main.cpp` versus the launcher), so they can share a wave. If two phases of a wave must edit the same file, the later one rebases on the earlier.

## Where Jason is needed

Phases 1, 3, 5, 7 and 8 end with a console run (protocol: [hardware-test-protocol.md](./hardware-test-protocol.md)). A phase's code can be merged as **unverified** before the run; the phase is **closed** only after Jason posts the run result. Wave N+1 may start on unverified code only when its phase file says so.

## Open decisions (ask Jason, recommended answer first)

| # | Question | Recommended |
|---|---|---|
| D1 | SD card folder location | The SD card exists: `/data/prosperoeden/user/sdmc` (Eden's own). Recommended: keep it, document it, create it only when a game opens it. A setting to move it to `<game files folder>/sd` (travels with the games on USB/M.2) is optional. Phase 2 asks Jason. |
| D2 | One SD card per console or per profile | Per console. ARC mods are big; profiles share them. |
| D3 | Which Smash version to target | **Answered by phase 0: 13.0.5**, the only version ARCropolis 4.1.1 supports. |

## Status

Update this table in the same change as the work.

| Phase | State | PR / result |
|---|---|---|
| 0 | Draft PR (findings written) | [findings.md](./findings.md) |
| 2 | Draft PR (D1 kept at `user/sdmc`) | [ProsperoEden #6](https://github.com/thatboialex/ProsperoEden/pull/6) |
| 1, 3 to 8 | Not started | |
