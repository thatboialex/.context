# Phase 4: Launcher status and switch

**Tier:** Sonnet. **Needs:** phase 2. **Wave:** 3 (beside phase 3). **Output:** code and docs in `thatboialex/ProsperoEden`.

## Goal

The player sees, in the Library, whether a game has Skyline, whether the SD card folder exists, and whether ARC mods are present, and can switch the lot off to play the game plain. No guessing why mods do not apply.

## Read first

`headless/mods.h` (`Kinds`, switches), `headless/patch_library.h`, `headless/settings_store.h` and `preferences.h` (per-game keys), `docs/MODS.md`, `docs/THEMES.md` only if text layout needs it. The launcher UI lives in `tools/launcher` and `examples/grid-launcher`; read how the Mods screen is built and copy it.

## Scope

- Detect Skyline in a mod: its `exefs/` holds `subsdk9` and `main.npdm` (extend the `Kinds` logic with a `kSkyline` flag; do not change existing flags).
- Game details line: `Skyline, 12 ARC mods` style text, using the same count wording as `1 of 2 mods on`.
- The existing Mods switch already turns a Skyline mod off; ensure a clear name (the mod's folder name) and a short note on its row.
- Count ARC mods: folders in `<SD card folder>/ultimate/mods/`. Cheap, bounded, off the main thread, no recursion (name count only).
- Strings for all 30 languages follow the repo's translation process; add English plus the key, and list the keys as "needs translation" in the PR (a Haiku thread fills them from the neighbours' pattern only if the repo's process says so).
- Host check in `mods_check.cpp` pattern.

## Out of scope

Installer, downloading, editing ARC mods in the launcher, per-ARC-mod switches (ARCropolis has its own manager; document the pointer instead).

## Done criteria

- Host check passes; mods list unchanged for games without Skyline.
- Library text appears for a Skyline game and not for others.
- `docs/MODS.md` has a short "Skyline and ARCropolis" paragraph that links to the guide planned in phase 8 (placeholder text allowed until then, labelled).
- CHANGELOG line added.

## Tests and review

Host check; Sonnet reviewer checks the UI follows the Mods screen's pattern and adds no work to the frame path. Console glance (screenshot from Jason) is nice, not required to merge.

## Result
State: Not started
