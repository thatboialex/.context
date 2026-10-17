# Phase 5: ARCropolis loads a custom character

**Tier:** Opus to diagnose; Sonnet for fixes the diagnosis specifies. **Needs:** phase 3 (closed). **Wave:** 4.

## Goal

With ARCropolis as a Skyline plugin and one custom-character ARC mod on the SD card folder, the character appears at character select and a match is playable.

## Read first

`findings.md` (questions 1, 2, 5, 9), `skyline-requirements.md`, `baseline.md`, the SD card section of `docs/INSTALL.md`.

## Scope

- Exact install layout for ARCropolis (findings 2: the hashes file is optional, not needed); write it into the protocol run so Jason can repeat it. Smash must report **13.0.5**; the version test below is ARCropolis's own dialog.
- Whatever ARCropolis needs that Skyline alone did not: file hooks on Smash's resource loading, large and many-small-file reads from the SD card folder, directory listing speed, path forms (`rom:/`, `sd:/`), case handling, network use when offline (its update check must fail fast; `auto_update` defaults to on, findings 2), and the `ninput::init()` branch ARCropolis takes because the load address is not one it knows (findings 9): if it hangs or crashes, that is the first thing to fix, and the options are listed there.
- Fix Gaps in Eden patches or `headless/`; same rules as phase 3.

## Out of scope

Performance tuning beyond "not unusably slow" (phase 7), diagnostics format (phase 6), UI (phase 4).

## Steps

1. Console run (protocol, phase 5): Skyline + ARCropolis, no ARC mods. Expected: game starts, ARCropolis log appears, menus work.
2. Run with one small, well-known custom-character ARC mod chosen by Jason (not committed anywhere).
3. Diagnose the first failure from the logs, fix, repeat. Record each in "Notes for the next session".
4. Test the failure modes that will happen to users: wrong Smash version, missing hashes, an ARC mod with a bad folder name, empty SD card folder. ARCropolis should say so itself; ProsperoEden must not hang or crash.

## Done criteria

- Character selectable and one match played (console, Jason).
- The four failure modes above do not hang or crash ProsperoEden; recorded in the protocol with the observed message.
- Vanilla and Skyline-only runs still match their earlier results.

## Tests and review

Host checks for any new logic; console runs for the rest. Opus reviewer for Eden patches, Sonnet for the rest.

## Result
State: Not started
