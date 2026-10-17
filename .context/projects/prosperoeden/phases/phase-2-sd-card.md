# Phase 2: SD card folder

**Tier:** Sonnet. **Needs:** phase 0. **Wave:** 2. **Output:** code and docs in `thatboialex/ProsperoEden`.

## Goal

The game sees an SD card (`sd:/`) whose contents are a folder the player controls and can fill over FTP, so ARCropolis finds `sd:/ultimate/mods/`. **Phase 0 found that Eden already mounts one at `/data/prosperoeden/user/sdmc`** (findings 5), so this phase is mostly decision, documentation and edge cases, not a new mount.

## Read first

`findings.md` (questions 5 and 8), `headless/storage_paths.h`, `headless/game_folder_vfs.h`, `headless/native_directory.h`, `docs/INSTALL.md` (layout diagrams), `docs/SAVES.md`.

## Scope

- Decide D1 (see the plan): keep `user/sdmc` (recommended) or move it with a setting to `<game files folder>/sd`. Ask Jason once. Whether Eden's path manager creating the folder at start breaks the rule that nothing is created behind the player's back (`mods.h`) is part of the decision.
- Only if D1 moves it: point Eden's SD card file system at the new place with the mechanism its neighbours use (README rule 3).
- Case: the host is case-sensitive, a Switch SD card is not. Decide from the protocol's case run whether guest paths under `sd:/` need folding (`mods.h` already does this for title folders); do not build it before the run shows a need.
- `SDMCFactory::GetSDMCFreeSpace` walks the whole SD folder (`eden:sdmc_factory.cpp:57-59`). Find out whether anything Skyline or ARCropolis does calls it; if so, cache or cap it.
- A host check (`headless/*_check.cpp` pattern) for the path logic.
- `docs/INSTALL.md`: the folder in the layout diagram (it is missing today), and what it is for. Plain words: "SD card folder". Mention that `atmosphere/contents/<title ID>/` inside it works like a mod.

## Out of scope

Launcher UI (phase 4), per-profile SD cards (D2), anything Skyline-specific.

## Steps

1. Implement the path decision and Eden hook per findings.
2. Add the host check: path resolved with the game files folder moved, with a missing folder, with mixed-case input.
3. Update INSTALL.md and the CHANGELOG ("next version" section).
4. Add the console run to the protocol if findings says behaviour cannot be proven on the host: a file written to `sd:/` by a small homebrew/NSP, or the Eden log line that names the mount.

## Done criteria

- `make test` passes with the new check (or the container's blocker is recorded).
- The SD card folder is created only when a game opens it, and a read-only drive does not crash.
- Docs and changelog updated.

## Tests and review

Host check as above. Sonnet reviewer checks: no waits on hot threads, folder never created behind the player's back, style of neighbours. Eden patch (if any) goes to an Opus reviewer.

## Result
State: Not started
