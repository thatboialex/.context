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
State: Draft PR
PR: https://github.com/thatboialex/ProsperoEden/pull/7 (stacked on [ProsperoEden #6](https://github.com/thatboialex/ProsperoEden/pull/6))
Changed:
- `headless/mods.h`: `kSkyline` (= 8, set beside `kCode` when `exefs/` holds `subsdk9` and `main.npdm`, exact names); `ArcModCount(sd_card)` (folders in `ultimate/mods/` by name, no recursion, cap 9999, -1 without the SD card folder).
- `headless/mods_check.cpp`: `check_skyline` (flags, one-of-two is not Skyline, other mods keep their flags, the Mods switch and the mod's own switch, the ARC count with and without the folder).
- Launcher (`pe/ui/services.hpp`, `launcher.hpp`, `library.cpp`, `home.cpp`, `runtime_models.hpp`, `eden_services.*`, `host/fake_services.*`): `Mod::skyline`, `Game::{skyline, skyline_on, arc_mods}`, `Home::last_*`, `Services::arc_mods()`, text from `Launcher::skyline_line`.
- Seven texts in `launcher.pot` and all 30 catalogs; `docs/MODS.md`, `docs/CHANGELOG.md`, `docs/RUNTIME_LAUNCHERS.md`.
Decisions:
- Text is a chip beside the update/DLC chip (Library details and home), not part of the "Mods" line: `Skyline, 12 ARC mods`; `Skyline, No ARC mods`; `Skyline, No SD card folder yet`; `Skyline off` (its mod or the game's Mods switch off). Shown only for a game with Skyline. ARC mods are only counted: ARCropolis has its own manager, the guide pointer is in `MODS.md`.
- The Skyline mod's row in the Mods list says `Skyline loader. Off: the game starts plain` in place of "Patch, Files". No new switch: the existing mod and Mods switches do it.
- The ARC folder is listed on the launcher thread, once per games refresh, `read_home` or `read_mods` (cached in `Launcher::arc_mods_`), never in a draw, and only for a game with Skyline on. This is the same thread that already lists the mods; the phase said "off the main thread" and there is no worker for it. Moving it into the game scan (`EdenServices::games`) is possible if a very large `ultimate/mods` makes the Library slow: **unverified** on the console.
- Translations: the repo's `strings.py check` fails on any untranslated text, so all 30 catalogs got a translation of the seven texts (written by Claude, not read by a native speaker; flagged in the PR). Nothing is left "needs translation".
- `docs/MODS.md` paragraph names the guide as planned and says Skyline and ARCropolis are the player's to supply (phase 8 replaces that with the link).
- `tools/launcher/generate-runtime-models.py` also wants to add `accent`, `memory_layout`, `update_notices`, `speed_limit` to `Preferences`; that drift is not this phase's, so that line was left as it was. Whoever touches Preferences next should regenerate.
Unverified: everything on the console and in the preview (no screenshot: the preview tool cannot fetch its dependencies in the cloud container); that `eden_services.cpp` and `mods_check.cpp` compile (no Eden headers; `check_skyline` compiled and run alone against `mods.h`; `library.cpp` and `home.cpp` pass `-fsyntax-only` with a stub GL header); `make test` not run. The chips drop when there is no room, so with a long update line the Skyline chip may be missing: a screenshot from Jason would show it.
Notes for the next session:
- Phase 8's guide must replace the placeholder sentence in `docs/MODS.md` ("A guide ... is planned").
- A game's Skyline mod counts as an ordinary mod in `1 of 2 mods on`.
- `Eden::SdCard::StateOf` was not used: the count is -1 when the folder is missing and the line says so; read-only is not shown (ARCropolis writes its config there, so a read-only drive would only show in the log line of phase 2).
