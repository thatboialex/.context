# Phase 2: SD card folder

**Tier:** Sonnet. **Needs:** phase 0. **Wave:** 2. **Output:** code and docs in `thatboialex/ProsperoEden`.

## Goal

The game sees an SD card (`sd:/`) whose contents are a folder the player controls and can fill over FTP, so ARCropolis finds `sd:/ultimate/mods/`. Skyline games on other emulators expect exactly this.

## Read first

`findings.md` (questions 5 and 8), `headless/storage_paths.h`, `headless/game_folder_vfs.h`, `headless/native_directory.h`, `docs/INSTALL.md` (layout diagrams), `docs/SAVES.md`.

## Scope

- Decide D1 (see the plan): default location and the setting to change it. Recommended: `<game files folder>/sd`, created on first need, never behind the player's back (`mods.h` has the rule).
- Point Eden's SD card file system at it. Prefer a small `tools/eden-patches/NN-*.diff` or an existing config path over new code; findings says which exists.
- Case-insensitive path matching if ARCropolis needs it and Eden's host file system does not give it (`mods.h` already does this for title folders).
- A host check (`headless/*_check.cpp` pattern) for the path logic.
- `docs/INSTALL.md`: the folder in the layout diagram, and what it is for. Plain words: "SD card folder".

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
