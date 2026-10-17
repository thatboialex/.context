# Phase 8: Guide, changelog, qualification

**Tier:** Haiku writes; Sonnet reviews; Jason qualifies. **Needs:** phases 5, 6, 7. **Wave:** 6. **Output:** `docs/SKYLINE.md` and edits in `thatboialex/ProsperoEden`.

## Goal

Someone with Smash, its update and their own copies of Skyline, ARCropolis and a custom character can get it working by following one page, with no help.

## Read first

All phase "Notes for the next session" blocks, `findings.md`, `baseline.md`, `docs/MODS.md` (voice and layout), `docs/INSTALL.md`.

## Scope

- `docs/SKYLINE.md` (facts to state, from findings): Smash must be exactly 13.0.5; Skyline's `main.npdm` is Smash's, so it is Smash-only; turn ARCropolis's `auto_update` off; the SD card folder is `user/sdmc` (or what phase 2 decided); mods for one game go in `roms/Smash/mods/<mod>/` with `exefs/` and `romfs/skyline/plugins/`; if two mods ship the same file the one whose folder name sorts first wins; Skyline opens an unauthenticated log port (whatever phase 3 decided). Contents: what it is, what you supply (and that ProsperoEden does not include or fetch it), the folder layout as a tree, step by step setup, memory setting, how to turn it off per game, what to attach to a report, the failure messages from phase 5 and what they mean. Voice of `MODS.md`: plain, no jargon unexplained.
- Links from README Guides table, `docs/MODS.md`, `docs/INSTALL.md` (SD card folder).
- CHANGELOG entry for the release, with the ARCropolis and Skyline versions it was qualified with.
- Fresh-setup qualification: Jason follows only the guide on a clean game folder (protocol phase 8 run).

## Out of scope

New features. A gap found here becomes a new phase file, not a fix in this PR.

## Done criteria

- The guide has every command, folder and setting; nothing needs chat history.
- Jason's clean run matches phase 5's result. Anything he had to guess is added to the guide.
- Plan status table all Closed; README status updated.

## Tests and review

Review: Sonnet reads the guide against the code and phase notes for wrong names. Link check. `validate-context`.

## Result
State: Not started
