# Plan: Skyline and ARCropolis on ProsperoEden

Status: Proposed, 2026-10-17. Requested by Jason. Code: none yet. How threads run it: [agent-workflow.md](./agent-workflow.md).

## Goal

Start Super Smash Bros. Ultimate from ProsperoEden with Skyline and ARCropolis loaded, so custom characters (ARC mods) work, with the player supplying every file. Setup is a short guide. A failure leaves a log that says which step failed.

Non-goals: online play with mods, bundling or downloading Skyline/ARCropolis/mods ([ADR 001](./decisions/001-user-supplied-skyline.md)), other Skyline games (kept possible, not tested), HDR/other big mod packs beyond a custom-character smoke test.

## Background (from memory, every item is checked in phase 0)

Mark these **unverified** until `findings.md` confirms them.

1. Skyline is installed to a game as `exefs/subsdk9` and a replaced `exefs/main.npdm` (more kernel capabilities and services). Plugins are `.nro` files in `romfs/skyline/plugins/`.
2. ARCropolis (`libarcropolis.nro`) reads the SD card: ARC mods in `sd:/ultimate/mods/<name>/`, its config in `sd:/ultimate/arcropolis/`, and a hashes file it needs to map file names. It also hooks the game's file loading.
3. Smash needs its update at a version ARCropolis supports (13.0.x). The update goes beside the game in `roms/<Game>/`.
4. Skyline patches game code at run time (inline hooks), loads NROs with the console's `ro` service and may use sockets for logs. The emulator must run those: exefs layering of `main.npdm`, `ro` service, SD card file system, memory permission calls and code-cache invalidation after a write.
5. ProsperoEden recognises `main`, `main.npdm`, `subsdk0..9` in a mod's `exefs/` (`headless/mods.h` `Kinds`). So step 1 may already load. What it lacks (to check): an SD card folder, setup guidance, status in the launcher, Skyline-specific diagnostics, memory defaults for a heavily modded Smash.
6. **Compile ahead** (block list) and the shader/JIT caches may assume game code does not change. A hooked game changes it. Phase 3 and 7 test this.

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
| D1 | SD card folder location | `<game files folder>/sd` so it travels with the games on USB/M.2; setting to override. Phase 2 confirms. |
| D2 | One SD card per console or per profile | Per console. ARC mods are big; profiles share them. |
| D3 | Which Smash version to target | The newest version ARCropolis supports, found in phase 0. |

## Status

Update this table in the same change as the work.

| Phase | State | PR / result |
|---|---|---|
| 0, 2 to 8 | Not started | |
| 1 | Draft PR: minimal boot check written, waiting for Jason's run | |
