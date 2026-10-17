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
State: Draft PR
PR: https://github.com/thatboialex/ProsperoEden/pull/6
Changed:
- `headless/sd_card.h` (new): the folder's path and state (missing, read-only, writable); looks only, never creates.
- `headless/sd_card_check.cpp` (new) and `headless/CMakeLists.txt`, `tools/build-headless-host.sh`: the host check, built and run with the others.
- `headless/CMakeLists.txt`: `eden_patch` on the derived `path_util.cpp` so the path manager's start-up pass skips `SDMCDir`.
- `headless/main.cpp`: one `[ProsperoEden] sd card:` log line after `CreateFactories`.
- `docs/INSTALL.md` (layout diagram, "SD card folder" section), `docs/CHANGELOG.md` ("Next version").
Decisions: D1 **kept** at `user/sdmc` (recommended option on the decision card; Jason had not answered when the PR was opened, so it is the default and moving it is still open).
- Lazy creation: Eden's `CreateFactories` runs at every game start and makes `sdmc/Nintendo/Contents/...`, so the folder exists after the first game start, not after app start. Opening the SD card read-write is what makes it (`RealVfsDirectory`'s constructor creates a missing folder when opened with write, `eden:vfs_real.cpp:406-408`). On a read-only drive that fails silently; `CreateDirectory` then returns null for `Nintendo/Contents/registered` and `placehld` (`eden:vfs_real.cpp:182-187`), and `RegisteredCache` returns early on a null directory (`eden:registered_cache.cpp:716-718`), so that step does not crash **[source only]**. Eden also makes an empty `atmosphere/contents/<title ID>/` for each game started (`GetSDMCModificationLoadRoot`, `eden:sdmc_factory.cpp:33-39`; `CreateDirectory` ignores the read-only open mode).
- Free space: `GetSDMCFreeSpace` is reached through `fsp-srv` free-space calls (`eden:filesystem.cpp:454,480`). A grep of Skyline `2eb226f` and ARCropolis `bf2005b` finds no call to any free-space or total-space function. Nothing built; the game itself might call it, which only a large-mods run shows (protocol, phase 5 row).
- Case: not built, as the phase said. The case run in the protocol decides.
Unverified: the folder is not made at app start and is made at game start (console); the `sd card:` log line; read-only drive start; that the `eden_patch` builds, because the cloud container cannot download dependencies. `make test` not run; `sd_card_check` compiled and run alone, as root and as a non-root user.
Notes for the next session:
- The path-manager change is an Eden source patch: per the phase it goes to an Opus reviewer.
- Phase 4 can read the folder state from `Eden::SdCard::StateOf(Eden::SdCard::Dir())`.
- Phase 3/5 find `sd:/` at `user/sdmc`; `ultimate/` goes directly under it. FTP to `/data/prosperoeden/user/sdmc/ultimate/mods/`.
- The changelog has no unreleased section upstream; "Next version" was added at the top and the maintainer renames it at release.
Review (Opus, Eden patch): **passed, no code change needed.** Checked against Eden `67bada77`:
- The `eden_patch` text occurs once in `src/common/fs/path_util.cpp` (`CreateEdenPaths`, line 94) and the CMake replacement applies (run with the repo's `eden_patch` function on the pinned source). `path.first` is the `EdenPath` key of `eden_paths`, and `EdenPath::SDMCDir` is in scope (`namespace Common::FS`). Not compiled: the container has no Boost or fmt.
- Only `CreateEdenPaths` made the folder before a game: `main.cpp:537,759` call it; `SetEdenPath` never creates (`path_util.cpp:303-309`); `DeriveSDSeed` only reads `Nintendo/Contents/private`; `CreateFactories` runs only at game start (`main.cpp:1534`, `eden:services.cpp:78`). The patch is inside `if(PS5_NATIVE)`, so host builds still make it at start (the host checks are unaffected).
- `sd_card.h` matches Eden's path (`UserDir()` + `SDMC_DIR` "sdmc", nothing calls `SetEdenPath(SDMCDir)`). The log line is on the main thread at game start: one `stat` and one `access`, no wait on a hot thread.
- `sd_card_check` rebuilt and run: passes as root and as `nobody` (read-only case).
- Nits left in the PR: on a read-only drive with no folder, the log says "does not exist", not "read-only"; INSTALL.md says `sdmc/atmosphere/...` where the glossary prefers "the SD card folder's".
