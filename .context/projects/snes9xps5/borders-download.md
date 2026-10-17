# Borders: the download from The Bezel Project (phase D)

Plan: [borders-plan.md](./borders-plan.md), why: [ADR 006](./decisions/006-borders-downloaded-from-the-bezel-project.md), the border system it feeds: [borders-built.md](./borders-built.md), [borders-ui.md](./borders-ui.md). Checked against snes9xPS5 branch `claude/borders-per-game-u4uc8t` (2026-10-17). Host-tested; nothing seen on a console.

## What exists

| Piece | Where | What it does |
|---|---|---|
| Name index | `ps5/frontend/data/snes-border-index.txt`, made by `ps5/tools/make_border_index.py` | The names of The Bezel Project's two SNES repositories, one per line: 1445 from `bezelproject-SNES` (`73dd9b0`, 2026-10-04; USA and World names), then after a `#@ SFC` line 2685 from `bezelproject-SFC` (`e8433ab`, 2026-05-19; Japan, Europe and the other regions; see below). Embedded in `fe_borderfetch.o` (`-DBORDER_INDEX_TXT`). Regenerate with `python3 tools/make_border_index.py` from `ps5/` (blobless clones; no picture is downloaded). |
| Name and address | `fe_borderfetch.h/.cpp` | `BezelNameFor(game)`: the cover matcher over the index (`PackMatcher`, new in `fe_covers`: indexes built once, used for the library and for single games). `BezelUrl(name)`: raw GitHub file, in the repository whose section lists the name; `SNES9X_BORDER_URL` replaces the base (host tests). `BorderWanted`, `MissingBorders`, `WriteWantedBorders`, `WritePriorityBorders`. |
| Request and live swap | `fe_border.cpp` | After `Choose`, under Auto, when the game got only `default.png` or the built-in frame: writes `borders/priority.txt` (one line) and starts a watcher (`ps5::BigThread`, 1 MiB stack) that stats `borders/games/<name>.png` twice a second, loads it on that thread and calls `SetBorder` only if the pick is still current (`s_gen`, `s_lock`). `Select`, `Reselect`, `Release` stop and join it; it gives up after 30 minutes or on a fresh `<name>.missing`. |
| Lookup of a downloaded file | `Choose` (`fe_border.cpp`) | New step after the pack matcher: the file named by the index (`"bezel name"` in the log), so a downloaded border is found even when the matcher, which sees only the folder, would not. |
| The helper | `fe_coverworker.h/.cpp` | `DownloadWorker`: one thread, one HTTP client, two lanes (covers, borders). Order: border priority, cover priority, other covers, library borders. Borders go to `borders/games/`, lists and `progress.txt` in `borders/`. Priority lists are now compared as text (a same-size list written in the same second was missed before). |
| Settings | `fe_menu.cpp`, `fe_shelf.cpp` | Shelf Settings: **Download all borders** (after Repair missing covers) writes `borders/wanted.txt`; its value shows `N left` from `borders/progress.txt`, `N queued`, `Waiting for the helper` (no progress file two minutes after pressing it), or `N fetched`. `border_download=0` in `snes9x-ps5.ini` (no row; the pause menu is full) turns Auto's downloads off. |
| Helper port | `ProsperoJailbreak.h` | 9089 to 9090, so a running older helper without the border lane is not used. |

Flow for a game: `LoadGame` > `Select` > `Choose` (own file, matcher over the pack, index name, `default.png`, built-in) > frame shown at once > if no own border: ask the helper, watch > file lands > loaded off the emulation thread > `SetBorder` (the renderer restores the backdrop for the new border).

## Japan and Europe: the second repository (2026-10-17)

Jason asked why Star Ocean and Seiken Densetsu 3 had no border. The Bezel Project keeps the SNES set in two repositories with no name in common: `thebezelproject/bezelproject-SNES` (USA, World; folder `retroarch/overlay/GameBezels/SNES/`) and `thebezelproject/bezelproject-SFC` (Japan, Europe and the other regions; folder `retroarch/overlay/GameBezels/SFC/`, same 1920x1080 palette PNG format, branch `master`). The first version of the index read only the first, so nearly every Japan-only game kept the built-in frame; the "Japan-only games mostly have none" below was this, not a gap in The Bezel Project. SFC has `Star Ocean (Japan)`, `Seiken Densetsu 3 (Japan)`, `Tales of Phantasia (Japan)`, `Live A Live (Japan)` and the rest. The index is now both lists; a name is downloaded from the repository whose section it is in. Same licence position as ADR 006 (none stated; nothing bundled).

Measured with the real matcher headers, `snes-nointro.tsv` and `snes-extra-crc.tsv` (the fuzzy name rescue in `fe_games.cpp` is not modelled):

| | SNES only | both |
|---|---|---|
| No-Intro names with a border | 2388 of 4268 | 4149 of 4268 |
| Japan releases | 290 of 1870 | 1837 of 1870 |
| Europe releases | 537 of 716 | 706 of 716 |
| English fan translations in the extra CRC table | 32 of 333 | 265 of 333 |

The 30 clean Japan names still without one are test and system carts (JRA PAT, Parame ROM Cassette, Super Game Boy, XBAND). 931 games that had a border now get a more exact one (a Europe or Japan release gets its own region's, not the USA one). Lost: five NES-conversion entries and three retranslations of games with a US release (`Breath of Fire II - Retranslation`, `Super Mario RPG - Relocalized`, `Ranma 1-2 - Treasure of the Red Cat Gang`): with the Japanese entries in the list the word matcher no longer picks between the USA and Japan names and returns none, as it is built to (the Ranma one had been given the wrong game's border).

Fan translations: Star Ocean's patches (DeJap, by CRC `A2B3B3FD`, `6BA9E08D`, `5CA81740`) and Seiken Densetsu 3's (Neill Corlett `5F14F4CA`, the Spanish and cheat versions) resolve by their CRC names; file names such as `Star Ocean (English)`, `Seiken Densetsu 3 [T+Eng v1.1]` resolve by words. The patch is also filed as `Secret of Mana 2`, which the matcher took for Secret of Mana plus a sequel number: `fe_artaliases.h` now maps it to `Seiken Densetsu 3 (Japan)` (covers too). `Star Ocean (J) T Eng, SDD1 Patched...` (CRC `B1D82240`) has an untagged comma list in its dump name and does not resolve by CRC name; its file name usually will. A ROM can always have its own: `/data/snes9x/borders/games/<ROM name>.png`.

Tests: `test_titlematch.cpp` (index sections, the two games by CRC name and file name, the alias, 1676 of 1718 clean Japan releases by name), `test_border_pack.cpp` (both addresses, a Japan game wanted from SFC), host group 36 (a Japan-only ROM in the library list; Europe has its own picture). Not tried on a console.

## Measured on the host (real matcher, real index, `snes-nointro.tsv`)

SNES repository only, before the above:

- Every one of the 1445 index names finds itself.
- Of 4268 No-Intro SNES names: 2388 get a bezel (1286 by exact name, 1102 by region, revision or translation matching, for example `Aladdin (Europe)` gets `Aladdin (USA)`), 1880 get none: 1469 Japan-only, 190 betas, prototypes, unlicensed or aftermarket, 221 other (mostly Europe-only titles with another US name, like `Terranigma (Spain)`).
- Matcher mistakes are the cover matcher's: `Super Puyo Puyo (Japan)` gets `Super Puyo Puyo 2`'s border and `Dragon Ball Z - Super Butouden (Japan) (Beta)` gets `Dragon Ball Z - Final Bout (USA) (Pirate)`.

## Tests

| Test | Covers |
|---|---|
| `make -C ps5 test-border` (`tests/test_border_pack.cpp`, 121 checks) | Index contents, address escaping, wanted rules (in pack, recent 404 for the same address only, 30 days), library list (one per bezel, own file skipped), list file format; Auto request, live swap, nothing put up after Release, another game, a setting change or a 404, `border_download=0`, Default frame, `off=1`; the index-name step. The matcher is a stand-in. |
| `make -C ps5 test-download-worker` (`tests/test_download_worker.cpp`, 16 checks) | The lanes against a fake HTTP client: order, destination folders, progress files, `.missing`, server error, offline pause, names with a folder part. Found the same-second priority list bug. |
| host group 36 in `tests/run-tests.sh` | The real app, helper and a local server: Auto asks, the helper downloads, the border goes up while the game runs, the next start finds it, `border_download=0`, Settings > Download all borders. |

## Not verified

On a console: the helper's TLS fetch of these files, the decode of a 1920x1080 palette PNG at game start and for a live swap, how the switch from the built-in frame to the game's border looks and costs, memory, and whether the installed helper really restarts on port 9090 after an update.
