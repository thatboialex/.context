# Borders: the Border row, the built-in frame, the docs

What the "UI and default art" thread built. Plan: [borders-plan.md](./borders-plan.md), spec: [borders-spec.md](./borders-spec.md). Code: `ps5/frontend/fe_menu.cpp` (row), `ps5/frontend/assets/borders/default.png` and `ps5/tools/make_default_border.py` (frame), `README.md` section "Borders around the picture" (user docs), `ps5/tests/run-tests.sh` group 35 (group 34 is the pack code's).

Status 2026-10-17: merged into snes9xPS5 `main` as `d009cb7` (#15), after the renderer (#17) and the pack code (#16). CI green; Codex review found nothing. Host-tested only: nobody has seen the frame on a PS5.

## The Border row

`S_BORDER` sits after `S_ASPECT` in `SettingRow`. Values Off, Auto, Default frame (`Settings::border` 0, 1, 2, key `border` in `snes9x-ps5.ini`); left and right wrap. The pause menu lists the same rows, and its reset row is `I_SET0 + S_UPDATES`, so inserting a row needed no other index change (checked: all 304 earlier host checks still pass). `ChangeSetting` ends in `emu::ApplySettings()`, which reselects the border in the running game. Off is the default until Jason has seen it on the console.

## The built-in frame

`default.png` is an original console cabinet, not copied from anywhere: graphite body, sloped bevel around a rounded glass opening, speaker grille (left), two dials and a power light (right), an empty name plate (bottom left). 1920x1080 palette PNG with `tRNS` (253 entries, the same kind of file as The Bezel Project's), 43 KB.

- **Regenerate:** `python3 tools/make_default_border.py` from `ps5/` (numpy and Pillow; the build does not run it, the PNG is checked in). Same bytes on every run with the same Pillow. The script reads the file back and fails if the size, the format, the opening or the 150 KB limit is wrong. `--preview out.png` writes the frame over a test picture to look at.
- **The opening** is exactly x 261, y 21, 1398x1038 (alpha below 16 inside, found by the flood fill in the spec), with 36 px rounded corners.
- **Where the soft edge is:** just *outside* the opening, 6 px, alpha 17 up to 255. A pixel with alpha 16 or more is not part of the window, so a soft edge cannot lie inside the window the way the spec's text pictures it: it stays in the backdrop (on black) and the picture meets it. What the renderer blends per frame for this frame is only the four corner pieces inside the window's box: 1124 pixels, not tens of thousands. A hand-made border with an edge of alpha 1 to 15 inside the window would add to that, nothing more.
- **Palette:** 214 entries for the dark cabinet, 24 for the few bright details (pointers, light) quantised apart so the dark colours cannot take theirs, 14 alpha steps of the lip colour for the soft edge, one clear entry. No dithering (it would blow the file size up).

## Docs and tests

- The user docs are in the main `README.md`, not `ps5/README.md` (that file only points there): the three values, the folder, naming, PNG format, `.ini` keys (`window=`, `off=1`), the Bezel Project drop-in (files are the user's own copy; nothing is bundled or downloaded), the built-in frame and how to replace it.
- Group 35 in `run-tests.sh` uses a generated blue border with the window clear (`mkborder`) and the red test ROM. It checks: Off draws nothing even with a pack present; Auto shows the border and the picture edge to edge (nine pixels around the window), the pause menu darkens both to a quarter (red 255 to 63), Resume restores them; Default frame and Auto with no pack show the built-in frame (colour read from `default.png`); the Settings row and the pause menu row, with wrap both ways, and a change in the pause menu showing at once.

## Not verified

How the frame looks at 1080p and on the 720p fallback; whether the dials, grille and name plate look right on a TV (only seen as a PNG over a test picture); cost per frame with the shaders.

## Open

- Whether Jason likes this frame. It is easy to change in the script; the name plate is empty on purpose.
- Flip the default to Auto only after Jason has used it on the console.
