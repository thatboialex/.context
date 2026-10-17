# Plan: borders around the picture

Status: Built and on snes9xPS5 `main`, 2026-10-17 (seam #14, renderer #17, pack and lookup #16, UI and built-in frame #15); host-tested, nothing seen on a console. What the threads built beyond this plan: [borders-built.md](./borders-built.md), [borders-ui.md](./borders-ui.md). Requested by Jason ("similar to how SwanStationPS5 has the PS1 borders and graphics while playing a game"). Formats and code seams: [borders-spec.md](./borders-spec.md). Key decision: [ADR 005](./decisions/005-borders-baked-into-the-surface.md).

## Goal

While a game runs, the area around the SNES picture shows art: a frame the game or the console would have had, instead of black bars. The picture sits in a window of the art. It must cost the emulation nothing measurable, work with every shader and aspect ratio, and be easy to extend by dropping files on the console.

Non-goals (first release): downloading art, animated borders, art drawn over the picture (except the corners and edge pixels of the glass inside the window), changing the shelf, a border editor.

## What SwanStationPS5 does (read 2026-10-17, commit `f7bea88`, GPL: design only, no code copied)

| Fact | Detail |
|---|---|
| Art source | The Bezel Project (`thebezelproject/bezelproject-PSX`, `GameBezels/PSX/<Redump name>.png`). The app ships only a name list (`assets/bezels-index.txt`, 8550 names), not the pictures. |
| Delivery | On first need the best name match is downloaded once into `<root>/art/bezels/<game id>.png` (`src/art.c`, one download at a time, `.part` then rename). The bezel kind downloads even when cover downloads are off. |
| Format | 1920x1080 PNG with alpha. A clear 4:3 window; the picture shows through it. |
| Draw | `src/bezels.c`: the game picture is drawn full height at 4:3 first, then the bezel texture is drawn over the whole screen with alpha (`plat_draw_texture`, tinted by the pause dim). The old border (black, backdrop, TV) is skipped when a bezel is shown. |
| When | Only for a 4:3 picture: a widescreen picture (16:9 aspect or the widescreen hack) goes without it. |
| Scope | One setting, Settings > Display > "Game artwork border", off by default. No per-game setting, no folder to drop files in. |
| Other borders | A black, "current background" or procedural "TV frame" border (`border=0/1/2`) for when there is no bezel. |

What carries over: PNG with alpha, a clear window, name-matched art per game, off by default, a plain fallback. What does not: its renderer is a GPU texture draw per frame (ours is a CPU surface, below), and it downloads on demand (ours does not in the first release, see "Art source").

## What snes9xPS5 does today (read at `b2157b3`)

| Fact | Where |
|---|---|
| The picture is drawn by the CPU into one linear 1920x1080 `A8B8G8R8` surface, then only the **changed rectangle** is swizzled into the scan-out buffer and flipped. There is no GPU path. A 1280x720 scan-out exists for low memory; the surface stays 1080p. | `ps5/coreorbis/orbis-shims/ProsperoVideo.h`, `Present` / `TileBlit` |
| `DrawSnes` fills the whole surface black when the picture's size, aspect, scanlines or shader changes, then writes only the picture rectangle on every other frame and returns that rectangle for `Present`. | `ProsperoVideo.cpp`, `DrawSnes` |
| The picture rectangle comes from `ComputeRect`: 4:3 (1440x1080), 8:7 at full height, largest integer multiple, or stretch 1920x1080. Screen-sized only. | `ProsperoVideo.cpp`, `ComputeRect` |
| Shaders (`ps5crt::Render`, CPU, multi-threaded) write into the picture rectangle only. | `ProsperoCrt.h` |
| After a menu or the cheat screen the app fills the surface black and calls `InvalidateSnes()`, so the next frame repaints everything. The pause menu paints the last frame, then darkens the whole surface twice. | `main-boot.cpp`, `fe_menu.cpp` (`PauseMenu`) |
| The OSD text is drawn by the core inside the picture. | `fe_emu.cpp` `Osd` |
| Settings are keys in `snes9x-ps5.ini`; rows in `SettingsMenu` (also shown in the pause menu) call `emu::ApplySettings()`. | `fe_settings.cpp`, `fe_menu.cpp` |
| Covers: names matched by words and the ROM header (`MatchArtName`, `fe_artmatch.h`), art lives in `/data/snes9x/covers`, downloads run in the helper payload (the sandboxed app cannot). | [box-art-lookup-plan.md](./box-art-lookup-plan.md) |
| PNG decoding is `stb_image` with `STBI_ONLY_PNG`, today only inside `fe_covers.cpp` (file-local). | `fe_covers.cpp` |

What the Bezel Project SNES set looks like (`thebezelproject/bezelproject-SNES`, checked 2026-10-17): 1445 bezels in `retroarch/overlay/GameBezels/SNES/<No-Intro name>.png`, each 1920x1080, 8-bit palette PNG, 40 to 380 KB. Five samples (Super Mario World, Chrono Trigger, F-Zero, Street Fighter II Turbo, Zoop) have the same clear window: x 261 to 1659, y 20 to 1058, about 1398x1038, ratio 1.347, with a soft semi-transparent edge inside it. The `.cfg` beside each only names the PNG; it carries no window coordinates. The repository has no `LICENSE` file (only `README.md` and `.gitattributes`), so redistribution rights are not stated.

## Design

The thing that makes this cheap: because only the picture rectangle changes from frame to frame, the border is **painted once into the surface** and never touched again, exactly where the black fill goes today ([ADR 005](./decisions/005-borders-baked-into-the-surface.md)).

1. A **border** is a 1920x1080 PNG with alpha. The **window** is its clear hole, found automatically (or given in a small `.ini`). The picture is drawn **inside the window** with the player's aspect setting.
2. When a border is loaded, the app builds a **backdrop** (the PNG composited on black, window black) and a short list of **rim** pixels (alpha 1 to 255 inside the window's box: rounded corners and any edge pixel the window search did not call clear).
3. `DrawSnes` copies the backdrop into the surface where it now fills black, draws the picture as before, then blends the rim pixels over the picture. Per frame that is the rim only: 1124 pixels for the built-in frame (four corner pieces), more for art whose edge has pixels of alpha 1 to 15 inside the window; the pack code refuses a border with over 400000 rim pixels. Everything else is untouched and not re-sent to the scan-out buffer. Cost on a console not measured.
4. The border is chosen when a game loads and when the setting changes, never on a worker the emulation waits for.

### Defaults chosen (change them by saying so)

| Topic | Choice | Why |
|---|---|---|
| Setting | `Border`: Off, Auto, Default frame. **Default Off** in the first release. | The window (about 1398x1038) is 3% smaller each way than the 1440x1080 picture; nothing on the console has been seen yet (README rule 8). Flip the default to Auto in a later change once Jason has used it. |
| Where art lives | `/data/snes9x/borders/` only: `default.png`, `games/<name>.png`, optional `.ini` beside them. | Covers and settings stay on `/data/snes9x` (README rule 3). Drop-in works over FTP or the console's file tools. |
| Names | A game's border is found the way its cover is: the ROM's own file name first, then the cover matcher over the file names in `games/` (No-Intro name, aliases, CRC name, spelling, words, ROM header). | One matcher, same results as the shelf, translations and renamed ROMs work. |
| Bezel Project art | Drop-in compatible: copy its `GameBezels/SNES/*.png` into `games/`; its `.cfg` files are ignored. | Same file names, same 1920x1080, nothing to convert. |
| Built-in art | One original frame ("TV"), drawn by `ps5/tools/make_default_border.py`, embedded in the ELF, under 150 KB. Original work, so it carries the repo's MIT licence. | Shipping the Bezel Project pictures is not allowed to be assumed (no licence stated). |
| Picture in the window | The Aspect setting applies inside the window. A shape within 3% of the window's own ratio fills it; any other shape is centred on black. | The Bezel Project window is 1.347, a hair off 4:3; filling avoids slivers. |
| Wrong art | A PNG that is not 1920x1080, has no clear centre or a window under 480x270 is refused with a log line, and the next choice applies (`default.png`, then built-in). | A bad file must never blank the game. |
| Memory | About 9 MB while a border is active (backdrop plus rim). If the allocation fails the border turns off and the game runs. | Same low-memory care the 720p scan-out already takes. |

### Art source: no downloads in the first release

SwanStationPS5 fetches The Bezel Project art on demand. Here the app cannot download after its jailbreak; the helper payload does it for covers (`fe_coverworker.cpp`, `covers/wanted.txt`). A border download would be a second queue in the helper plus a name index of 1445 entries, and the licence question above. It is therefore a separate, later phase (D below), switched on only by Jason. Until then Jason drops the pack in `/data/snes9x/borders/games/`.

## Work split: three threads in parallel

Everything shares two small interfaces, written down in [borders-spec.md](./borders-spec.md) ("Seams"). Thread A lands them first as a tiny seam change that behaves exactly like today (no border): the `Border` type and `SetBorder` stub, the `border` setting key, an empty `fe_border.cpp` with its Makefile line, and a placeholder `default.png`. B and C start from it and never wait on each other's internals.

| Thread | Owns (only this thread edits these) | Delivers |
|---|---|---|
| **A: renderer** | `ps5/coreorbis/orbis-shims/ProsperoVideo.h`, `ProsperoVideo.cpp`; `ps5/tests/test_border_render.cpp` (new); the seam in `fe_settings.h/.cpp` (`border` key, nothing else) | `ps5video::Border`, `SetBorder`, window-aware `ComputeRect`, backdrop restore, rim blend, no cost when no border. Host tests for geometry, backdrop and rim. |
| **B: pack and lookup** | `ps5/frontend/fe_border.h/.cpp`, `fe_borderimg.cpp` (new, own `stb_image` copy), `fe_covers.h` plus a one-function export in `fe_covers.cpp`, the `Apply`/`Release` calls in `fe_emu.cpp`, the Makefile lines for its files, `ps5/tests/test_border_pack.cpp` (new) | Pack folder scan, per-game match, PNG decode, window detection, rim and backdrop build, `.ini` overrides, fallbacks and log lines. Host tests with generated PNGs. |
| **C: UI and default art** | `ps5/frontend/fe_menu.cpp` (the `Border` row only), `ps5/frontend/assets/borders/` (new), `ps5/tools/make_default_border.py` (new), `ps5/README.md`, the `.context` notes for the finished feature | The settings row (also in the pause menu), the built-in frame and its generator, user docs ("how to add a border pack"), a host render test group in `ps5/tests/run-tests.sh` that shows a border at 1080p. |

Merge order: A's seam first (minutes), then A, B and C in any order. Shared files, resolved: the seam commit is the one exception to the ownership table (A alone adds the stub `fe_border.*`, its Makefile line and the placeholder `default.png`); once it is on `main` the Makefile is B's (C adds only an asset path B's `-D` flag already names) and A touches none of those files; `fe_settings.*` is A's for the seam and C only reads it; `run-tests.sh` is appended to by A, B and C, each at the end with its own group comment and number (read the last one first; on a conflict keep both blocks).

Phase D, after A to C merge and Jason confirms it on the console: on-demand download through the helper (a `borders/wanted.txt` queue, `snes-border-index.txt`, `fe_coverworker.cpp`), off by default, after the licence of the Bezel Project art is settled with Jason. Not started until he asks.

## Acceptance

1. Border Off: the picture, timing and `make -C ps5 test` output are unchanged from `main`.
2. Border Auto with a pack: a game with a matching `games/<name>.png` shows it, the picture exactly fills the window, soft edges blend, in every Aspect and with every shader.
3. A translated or renamed ROM finds the original game's border the way its cover is found.
4. A broken, wrong-size or missing file falls back (`default.png`, then built-in) with a one-line log message; the game never fails to start.
5. Pause menu: the border is visible and darkened with the picture; Resume restores it without artefacts.
6. A frame costs nothing outside the picture rectangle: the `Present` rectangle for a normal frame is the picture rectangle, as today (host test asserts it).
7. `make -C ps5 test` and `make -C ps5 ps5` pass. Console behaviour (look at 1080p, frame time with the shaders) stays labelled unverified until Jason confirms it.

## Risks

| Risk | Handling |
|---|---|
| A game that switches between 256 and 512 pixel modes repeats the backdrop copy (about 8 MB, an estimated 1 to 2 ms) every switch. | Restore the backdrop only when the picture rectangle or the border changes, not on every source-size change (spec, "Renderer"). |
| Art with a window that is not centred or not a rectangle. | Window = bounding box of the clear region around the centre; anything outside it but clear shows black. `.ini` `window=` overrides. |
| `stb_image` defined in two files. | B's copy is `STB_IMAGE_STATIC` in its own file, as `fe_covers.cpp` does; no shared symbols. |
| Palette PNGs with a `tRNS` chunk (the Bezel Project's) decoded wrongly. | stb handles it; B's test generates one. |
| 720p scan-out: the border is downscaled with the picture. | No work needed; same path as today. |
| Licence of third-party art. | Nothing third-party is bundled; the drop-in pack is the user's own copy. |
