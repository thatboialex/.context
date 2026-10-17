# Phase 0: Findings

**Tier:** Sonnet (Opus only for the item marked escalate). **Needs:** nothing. **Wave:** 1. **Output:** `projects/prosperoeden/findings.md` in `.context`; no code.

## Goal

Replace the plan's "from memory" background with verified facts, each with a source (file and line, or URL), so later phases do not guess.

## Read first

- The plan's Background section.
- ProsperoEden: `UPSTREAM.json`, `tools/deps.py`, `tools/eden-patches/*.diff`, `headless/mods.h`, `headless/game_folder_vfs.h`, `headless/storage_paths.h`, `docs/MODS.md`, `docs/INSTALL.md`, `docs/BUILDING.md`.
- Eden source at the pinned commit (`UPSTREAM.json` `eden_commit`; archive `eden-67bada77.tar.gz` via `make deps`, or from `https://github.com/eden-emulator/mirror` at that commit if the container can reach it). Put it in a scratch directory, not in either repo.
- Skyline (`skyline-dev/skyline`) and ARCropolis (`Raytwo/ARCropolis`) READMEs, wiki and releases.

## Questions to answer (one section each in findings.md)

1. Which Smash update version does the newest ARCropolis release support? Title IDs of base, update, DLC.
2. Exact files Skyline and ARCropolis install and where (exefs, romfs, SD card), and what `main.npdm` changes (services, kernel capabilities, memory).
3. Does Eden's ExeFS patching honour a replaced `main.npdm` and `subsdk9` from a mod? Cite the code path. Does `headless/mods.h` and the derived `bis_factory.cpp` hand them over?
4. Which services and SVCs does Skyline use at start (`ro`/`ldr:ro`, `fsp-srv` SD card, sockets, `set:sys`, memory-permission and code-memory SVCs)? For each: implemented in Eden at the pin, stubbed, or missing.
5. Does Eden mount an SD card file system, and at which path (`sdmc`)? How does ProsperoEden's `user/` directory and `storage_paths.h` place it? What does ARCropolis need from `sd:/` (case sensitivity, directory listing, big files)?
6. Self-modifying code: when a guest writes to its code and runs `ic ivau`/`dc cvau`, does Eden invalidate the JIT block (dynarmic instruction-cache hook)? Does ProsperoEden's JIT page table / alias code (`jit-alias.cmake`, `jit-permissions.inc`) allow it? Does "Compile ahead" or the block list assume code does not change?
7. Does the container run `make test`? What does it need? Record the exact command that works, or what blocks it.
8. What do Smash plus mods need in memory? (Smash's requirement, ARCropolis's resident size, ProsperoEden's `memory_layout` options.)

## Steps

1. Create the scratch directory, extract Eden, run the greps the questions imply.
2. Write findings.md: one section per question, answer first, then evidence, then "Gap" (what ProsperoEden or Eden lacks) and "Phase" (which phase closes it).
3. Update the plan: fix any wrong Background item, adjust phases and open decisions D1 to D3. If a finding changes a phase's scope, edit that phase file's Scope.
4. If findings show the mods path cannot carry Skyline, edit ADR 001 to Superseded and say what replaces it.

## Done criteria

- Every question has an answer with evidence, or "not determinable from source" and the run that would settle it (added to hardware-test-protocol.md).
- Every Gap names a phase. No Gap is unowned.
- Plan Background has no unverified items left, or marks the remaining ones.

## Tests and review

No code. Reviewer (Sonnet, fresh thread) spot-checks five claims against the source, picking the ones that set the most work. Run `validate-context`.

## Escalate to Opus

Question 6 if the cache invalidation path is unclear after one read of Eden's `arm_dynarmic_64.cpp` and ProsperoEden's JIT files.

## Result
State: Not started
