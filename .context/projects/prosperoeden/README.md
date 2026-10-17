# ProsperoEden: project context

Entry point for any thread working on `thatboialex/ProsperoEden`. Read this file first, then load only the file your task needs.

## What it is

An unofficial PS5 port of the Eden Switch emulator (title ID PPSA99008), by BlackBearReloaded. `thatboialex/ProsperoEden` is Jason's fork; upstream is `blackbearreloaded/ProsperoEden`. PS5 frontend and launcher in `headless/`, build tools in `tools/`, Eden patches in `tools/eden-patches/`, user guides in `docs/`. Eden itself is fetched at the commit pinned in `UPSTREAM.json`; it is not in the repo.

## Current goal

Run **Skyline** (code-injection framework) and **ARCropolis** (Smash Ultimate mod loader, a Skyline plugin) so Jason can play Super Smash Bros. Ultimate (title ID `01006A800016E000`) with custom characters. Plan: [skyline-arcropolis-plan.md](./skyline-arcropolis-plan.md).

## Repos

| Repo | Role |
|---|---|
| `thatboialex/ProsperoEden` | Code and user docs (`docs/`). Default branch `main`. |
| `thatboialex/.context` | Plans, specs, ADRs and status for it. Plans and notes go here, not in the code repo. |

## Hard rules

1. ProsperoEden does not include or download games, keys, firmware, mods, Skyline, ARCropolis or hashes. The player supplies them ([docs/MODS.md](https://github.com/thatboialex/ProsperoEden/blob/main/docs/MODS.md) already says so). Never commit or fetch Nintendo content or mod binaries. No piracy help in docs.
2. Prefer the existing mods path (`roms/<Game>/mods/<mod>/{exefs,romfs}`, `headless/mods.h`) over new mechanisms. Add a mechanism only when `findings.md` shows the mods path cannot do it.
3. Changes to Eden are either a numbered diff in `tools/eden-patches/NN-name.diff` (4 so far, applied by `tools/prepare-build.sh`) or a string replacement of an Eden source in `headless/CMakeLists.txt` (`eden_patch(...)`, `write_derived(...)`, which stops the configure when the pin changes). Use the one the neighbouring change uses; keep it small. Do not vendor Eden.
4. No waits on the emulation, video or audio threads for new file or log work.
5. Match the neighbouring file's style and licence header (`// SPDX-License-Identifier: GPL-3.0-or-later` in `headless/`). New user-facing behaviour gets a section in the matching `docs/*.md` and a `docs/CHANGELOG.md` line under the next version.
6. Every code change passes `make test` on the host when the cloud session can run it; say plainly when it cannot.
7. A cloud session cannot run the console. Label console behaviour **unverified** until Jason confirms it on hardware with [hardware-test-protocol.md](./hardware-test-protocol.md).
8. Do not guess what Skyline or ARCropolis do. Phase 0 recorded what the sources show in `findings.md`; later phases cite it, and a finding that turns out wrong is corrected there.

## Terms (use exactly these)

| Term | Meaning |
|---|---|
| **Skyline** | The code-injection framework for Switch games: `subsdk9` plus a replaced `main.npdm`, in the game's exefs. Loads plugins. Not the old Skyline Android emulator. |
| **ARCropolis** | The Smash Ultimate mod loader. A Skyline plugin (`libarcropolis.nro`). Not "Acropolis". |
| **Plugin** | An `.nro` Skyline loads at game start. |
| **Smash** | Super Smash Bros. Ultimate, `01006A800016E000`. Needs its update (13.0.x, see findings). |
| **ARC mod** | A folder under the SD card's `ultimate/mods/` that ARCropolis applies (a character, a stage, a music change). Not a ProsperoEden **mod**. |
| **Mod** | ProsperoEden's word: a folder under `roms/<Game>/mods/` (exefs and romfs). Skyline and its plugins are delivered as one. |
| **SD card folder** | The folder ProsperoEden shows the game as the console's SD card (`sd:/`). Not "sdmc" in user text. |
| **Wave** | The phases that can run at the same time. |

## Files

| File | Load when |
|---|---|
| [skyline-arcropolis-plan.md](./skyline-arcropolis-plan.md) | You start any phase: goal, background, phase list, order, status table. |
| [agent-workflow.md](./agent-workflow.md) | You are a thread picking up a phase: how to work, tiers, review, handoff. |
| [hardware-test-protocol.md](./hardware-test-protocol.md) | You need Jason to test on the console, or write a test step. |
| [phases/](./phases/) | One file per phase, `phase-N-name.md`. Read only yours. |
| [decisions/001-user-supplied-skyline.md](./decisions/001-user-supplied-skyline.md) | You question why Skyline and ARCropolis are not bundled. |
| [findings.md](./findings.md) | You need a verified fact about Eden, Skyline or ARCropolis. |
| [emulator-reference.md](./emulator-reference.md) | You want to see how Ryujinx and yuzu (via Eden) ran Skyline and ARCropolis. |

## Status

| Date | State |
|---|---|
| 2026-10-17 | Plan and phase docs written (draft PR in `.context`). No code started. |
| 2026-10-17 | Phase 0 done: `findings.md`. Smash 13.0.5 only; Eden already supports what Skyline needs; the SD card already exists; `make test` cannot run in the cloud container. |
| 2026-10-17 | Phase 2 draft PR: SD card folder kept at `user/sdmc`, documented, made when a game starts ([ProsperoEden #6](https://github.com/thatboialex/ProsperoEden/pull/6)). |
