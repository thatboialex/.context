# Borders: the download from The Bezel Project (phase D)

Plan: [borders-plan.md](./borders-plan.md), why: [ADR 006](./decisions/006-borders-downloaded-from-the-bezel-project.md), the border system it feeds: [borders-built.md](./borders-built.md), [borders-ui.md](./borders-ui.md). Checked against snes9xPS5 branch `claude/borders-per-game-u4uc8t` (2026-10-17). Host-tested; nothing seen on a console.

## What exists

| Piece | Where | What it does |
|---|---|---|
| Name index | `ps5/frontend/data/snes-border-index.txt`, made by `ps5/tools/make_border_index.py` | The 1445 names The Bezel Project's `bezelproject-SNES` has (`73dd9b0`, 2026-10-04), one per line. Embedded in `fe_borderfetch.o` (`-DBORDER_INDEX_TXT`). Regenerate with `python3 tools/make_border_index.py` from `ps5/` (a blobless clone; no picture is downloaded). |
| Name and address | `fe_borderfetch.h/.cpp` | `BezelNameFor(game)`: the cover matcher over the index (`PackMatcher`, new in `fe_covers`: indexes built once, used for the library and for single games). `BezelUrl(name)`: raw GitHub file; `SNES9X_BORDER_URL` replaces the base (host tests). `BorderWanted`, `MissingBorders`, `WriteWantedBorders`, `WritePriorityBorders`. |
| Request and live swap | `fe_border.cpp` | After `Choose`, under Auto, when the game got only `default.png` or the built-in frame: writes `borders/priority.txt` (one line) and starts a watcher (`ps5::BigThread`, 1 MiB stack) that stats `borders/games/<name>.png` twice a second, loads it on that thread and calls `SetBorder` only if the pick is still current (`s_gen`, `s_lock`). `Select`, `Reselect`, `Release` stop and join it; it gives up after 30 minutes or on a fresh `<name>.missing`. |
| Lookup of a downloaded file | `Choose` (`fe_border.cpp`) | New step after the pack matcher: the file named by the index (`"bezel name"` in the log), so a downloaded border is found even when the matcher, which sees only the folder, would not. |
| The helper | `fe_coverworker.h/.cpp` | `DownloadWorker`: one thread, one HTTP client, two lanes (covers, borders). Order: border priority, cover priority, other covers, library borders. Borders go to `borders/games/`, lists and `progress.txt` in `borders/`. Priority lists are now compared as text (a same-size list written in the same second was missed before). |
| Settings | `fe_menu.cpp`, `fe_shelf.cpp` | Shelf Settings: **Download all borders** (after Repair missing covers) writes `borders/wanted.txt`; its value shows `N left` from `borders/progress.txt`, `N queued`, `Waiting for the helper` (no progress file two minutes after pressing it), or `N fetched`. `border_download=0` in `snes9x-ps5.ini` (no row; the pause menu is full) turns Auto's downloads off. |
| Helper port | `ProsperoJailbreak.h` | 9089 to 9090, so a running older helper without the border lane is not used. |

Flow for a game: `LoadGame` > `Select` > `Choose` (own file, matcher over the pack, index name, `default.png`, built-in) > frame shown at once > if no own border: ask the helper, watch > file lands > loaded off the emulation thread > `SetBorder` (the renderer restores the backdrop for the new border).

## Measured on the host (real matcher, real index, `snes-nointro.tsv`)

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
