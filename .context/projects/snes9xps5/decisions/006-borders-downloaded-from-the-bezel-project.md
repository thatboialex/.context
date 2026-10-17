# ADR-006: Per-game borders are downloaded from The Bezel Project; no picture is bundled

## Status
Proposed, 2026-10-17. Built in snes9xPS5 (see [borders-download.md](../borders-download.md)); host-tested, nothing seen on a console. Becomes Accepted when Jason has said that The Bezel Project is an acceptable source and has seen a downloaded border on the console. His calls are listed at the end.

## Context

Jason confirmed the built-in frame works and asked for borders specific to each SNES game, as SwanStationPS5 has for PS1 games (his message in the thread "Per-game SNES borders"). The border system of [ADR 005](./005-borders-baked-into-the-surface.md) already loads `/data/snes9x/borders/games/<ROM>.png` and finds renamed ROMs by the cover matcher; what was missing was the art, and phase D of the [plan](../borders-plan.md) (download) was held back over the licence of The Bezel Project's pictures.

**What SwanStationPS5 does** (read at `main`, 2026-10-17; design only, its code is GPL-3.0-or-later):
- `src/art.c`: the "bezel" kind downloads `https://raw.githubusercontent.com/thebezelproject/bezelproject-PSX/master/retroarch/overlay/GameBezels/PSX/<name>.png` on first need, into `<root>/art/bezels/`.
- It ships only the list of names (`assets/bezels-index.txt`, 8550 lines), never a picture.
- Its README credits and `THIRD_PARTY_NOTICES.md` do not name The Bezel Project; only the header comment of `src/art.c` does. Its other art source is the same kind: `xlenore/psx-covers`, "downloaded at runtime by serial; not bundled", licence "see that repository".
- So SwanStationPS5 relies on the user's console fetching from GitHub, with no licence for the pictures. Our covers do the same with libretro-thumbnails (README, Credits: "downloaded on the console, not included").

**Licences of the SNES bezel sets** (checked 2026-10-17 from the repositories' files):

| Source | What it has | Licence stated | Per-game SNES borders |
|---|---|---|---|
| `thebezelproject/bezelproject-SNES` (`73dd9b0`, 2026-10-04) | 1445 bezels, 1920x1080 palette PNG, `retroarch/overlay/GameBezels/SNES/<No-Intro name>.png`: our drop-in format | None. The root holds `.gitattributes`, `README.md`, `retroarch/`; the README says it is "a ZIP package compiled by The Bezel Project for use with their automatic downloader". The sibling repos (`BezelProject`, `bezelproject-PSX`) have no LICENSE either. The pictures are built from the games' own art (Super Mario World's has Mario, Yoshi, the game logo and the Super Nintendo logo), so even a licence from the compilers would not cover the Nintendo art in it. | 1445 |
| `libretro/common-overlays` | `borders/img/snes-*.png`: `castlevania4`, `ct`, `demonscrest`, `lttp`, `metroid`, `smas`; 1920x1080 RGBA | CC BY 4.0 for the repository (`COPYING`). The same caveat: the art shows the games' pictures, which the contributor could not license. | 6 |
| `libretro/overlay-borders` | Generic system borders (NyNy77 and Nosh01 sets for SNES/Super Famicom, Vectrex) | MIT (`LICENSE`, "Copyright (c) 2017") | none, one border per system |
| Other SNES bezel packs (Orionsangel's, community packs on forums) | Searched: only forum threads, no repository with a licence file or terms | None found | not examined further |

No source is clearly redistributable for per-game SNES art. The only per-game set with a stated licence has six games.

## Decision

1. **The app carries names, not pictures.** `ps5/frontend/data/snes-border-index.txt` lists the names The Bezel Project has: 1445 from `bezelproject-SNES` (USA and World) and, after a `#@ SFC` line, 2685 from `bezelproject-SFC` (Japan, Europe and the other regions; added 2026-10-17, see the amendment under Consequences). `ps5/tools/make_border_index.py` reads them from the repositories' trees; no picture is downloaded to make it. Nothing from The Bezel Project's pictures is in the ELF or in our repositories.
2. **Source: The Bezel Project, as SwanStationPS5 does.** The console downloads `GameBezels/SNES/<name>.png` (from `bezelproject-SNES`) or `GameBezels/SFC/<name>.png` (from `bezelproject-SFC`, for the names in the index's SFC section) from `raw.githubusercontent.com` through the helper (the app cannot download after its jailbreak, same as covers), into `/data/snes9x/borders/games/<name>.png`, under the project's own name. These are the files the drop-in format already expects.
3. **When.** Border on Auto and the game has no border of its own (no own file, no match in the pack): the app finds the game's name in the index with the cover matcher (so a European release, a revision or a translation gets the original game's border), asks the helper for it first in line (`borders/priority.txt`), keeps the built-in frame up, and puts the border up when the file lands, from a watcher thread, never on the emulation thread. Settings > Download all borders asks for the whole library in the background (`borders/wanted.txt`, after the covers).
4. **Off switches.** Border Off and Default frame never touch the network. `border_download=0` in `snes9x-ps5.ini` turns downloading off for Auto. A 404 is remembered for 30 days (`<name>.missing`), as for covers.
5. **Nothing is asked of a game with a border of its own** (an own file, a match in the pack, an `.ini` with `off=1`).
6. **The helper listens on a new port (9090)** so that an older helper still running (which has no border lane) does not take the app's requests and silently never fetch.

## Alternatives considered

| Option | Why not |
|---|---|
| Bundle The Bezel Project's pictures (about 290 MB for all 1445, estimated) | No licence, and the art is the publishers'. We would be the redistributor. |
| Bundle the six `common-overlays` borders (CC BY 4.0) | Six games, the licence does not cover the games' art in them, and attribution in the ELF. Not worth the risk for 6 of 1445. |
| Draw original art for every game | Not feasible; the built-in frame is the original art we can ship. |
| Manual drop-in only (what shipped in ADR 005) | Works, but is not "out of the box", which is what Jason asked for. Still supported. |
| Our own mirror of the pictures | Makes us the redistributor, which is the problem. |
| List the repository through the GitHub API at run time | Rate limited and needs a token; a name list built in is exact and offline-safe. |

## Consequences

### Positive
- Per-game art for about 56% of the No-Intro SNES names (2388 of 4268, measured with the real matcher: 1286 by exact name, 1102 by region, revision or translation matching), nearly all USA and Europe releases; Japan-only games (1469 of the 1880 without) keep the default or built-in frame. **Amended 2026-10-17:** that read only `bezelproject-SNES`; the Japan, Europe and other regions' borders are in the sibling `bezelproject-SFC` (same format, no name in common, same licence position), and the index now lists both: 4149 of 4268, Japan 1837 of 1870. See `borders-download.md`.
- The user's own pictures still win; the drop-in works as before.
- One more lane in the helper's existing download loop; no new thread in the helper, no second TLS client.

### Negative
- Depends on a third-party repository staying at the same address; the index goes stale as it adds files (regenerate with the tool; a name removed from the repository ends as a 404 and a `.missing` marker).
- The pictures are 100 to 400 KB each: about 60 MB for a library of 300 games (estimate).
- The licence of the pictures is unstated. The project is in the same position as SwanStationPS5 and as our own covers; it is a risk Jason accepts or not (below).
- The cover matcher's mistakes become wrong borders (for example it maps `Super Puyo Puyo (Japan)` to the border of `Super Puyo Puyo 2`).

### Neutral
- A new setting key (`border_download`, on by default), a new folder file set under `borders/` (`wanted.txt`, `priority.txt`, `progress.txt`), and the helper's port changes from 9089 to 9090.

## Jason's calls

1. **Keep The Bezel Project as the source?** If not: flip the default of `border_download` to off in `fe_settings.h` (one line), or point `kBezelBase` in `fe_borderfetch.cpp` at a source he controls, or remove the index and the Auto download (the drop-in stays).
2. **Make Auto the default for Border?** It is still Off by default (as in ADR 005 and as SwanStationPS5's "Game artwork border"); the first Auto game downloads a picture, so it is his to switch on or to make the default.
3. Console behaviour, all unverified: the helper's TLS reaching `raw.githubusercontent.com` for these files (it does for covers), the decode time of a 1920x1080 palette PNG at a game start and on a live swap, the visible switch from the built-in frame to the game's border while it runs, and memory.
