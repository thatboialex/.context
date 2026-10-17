# Spec: borders around the picture

For the thread that writes the code. The why is in [borders-plan.md](./borders-plan.md) and [ADR 005](./decisions/005-borders-baked-into-the-surface.md). Paths are relative to the snes9xPS5 repo root; line numbers drift, re-check with `rg`. All new files start with `// SPDX-License-Identifier: MIT` and match the neighbouring file's style (tabs and Allman braces in `ps5/frontend` and `ps5/coreorbis`).

## Terms (use exactly these)

| Term | Meaning |
|---|---|
| **Border** | The art around the SNES picture. Not "bezel", "overlay" or "frame"; say "bezel" only when naming The Bezel Project. |
| **Border pack** | The folder `/data/snes9x/borders/` and what is in it. |
| **Window** | The clear hole of a border, as a rectangle of the 1920x1080 surface. The picture goes inside it. |
| **Backdrop** | The border composited on black, window black, kept as a ready-made surface copy. |
| **Rim** | The pixels of the border inside the window with alpha above 0: the soft edge or rounded corner of the glass. Blended over the picture every frame. |

## Seams (Thread A lands these first; B and C code against them)

Behaviour after the seam change is identical to `main`: no border is ever set. The seam commit is the one exception to the ownership table in the plan: A alone creates the stub `fe_border.h/.cpp`, their Makefile line and the placeholder `default.png` in it. Once it is on `main`, those files belong to B and C as the table says, and A touches none of them again.

`ps5/coreorbis/orbis-shims/ProsperoVideo.h` (A), inside `namespace ps5video`, after `Rect`:

```cpp
// One pixel of the border that lies inside the window: drawn over the picture every frame.
struct RimPixel
{
	uint32_t index; // y * kWidth + x
	uint32_t pre;   // 0xAABBGGRR with A = 0: each colour already multiplied by alpha / 255
	uint32_t inv;   // 255 - alpha
};
struct Border
{
	std::vector<uint32_t> backdrop; // kWidth * kHeight, 0xAABBGGRR, opaque. Window pixels 0xff000000, rim pixels already on black.
	Rect window;
	std::vector<RimPixel> rim; // every pixel inside `window` with alpha 1..255, sorted by index
};
// The border DrawSnes uses from the next frame on; null = none (today's black). Cheap: swaps a pointer.
void SetBorder(std::shared_ptr<const Border> border);
```

`ps5/frontend/fe_settings.h` (A, seam only): `int border = 0; // 0 Off, 1 Auto, 2 Default frame`, loaded and saved as `border=` (clamp 0 to 2) in `fe_settings.cpp`. Nothing else in these files changes.

`ps5/frontend/fe_border.h` (B; A's seam commit adds the header and an empty `fe_border.cpp` with the Makefile line, B fills it in):

```cpp
namespace fe::border
{
// Picks and loads the border for the game that was just loaded (its ROM path and the CRC the core computed), or clears it
// when the setting is Off. Calls ps5video::SetBorder. Never throws, never fails the game: a problem is one OrbisLog line.
void Select(const std::string& rom_path, uint32_t rom_crc);
// The setting changed while a game is loaded: re-pick for the same game. No-op with no game.
void Reselect();
// The game was closed.
void Release();
}
```

`ps5/frontend/assets/borders/default.png` (C's directory, but A's seam commit adds a placeholder so B's build has its `.incbin` target): a plain black 1920x1080 palette PNG with a clear 1398x1038 window at (261, 21). C replaces the file with the real frame; replacing is a modification, so it does not conflict.

`emu::LoadGame` calls `Select` after the ROM loaded and before `netfs_set_realtime(true)`; `emu::ApplySettings` calls `Reselect`; `emu::CloseGame` calls `Release` (all in `fe_emu.cpp`, B's edit).

## Pack layout and formats

```
/data/snes9x/borders/
  default.png          optional: used when a game has no border of its own
  default.ini          optional
  games/
    <name>.png         a border for one game
    <name>.ini         optional, same stem
```

**PNG.** Exactly 1920x1080, 8 bits per channel: RGBA, palette with `tRNS` (The Bezel Project's files are palette), or RGB. An RGB image is opaque everywhere, so it is accepted only with a `window=` in its `.ini` (otherwise "no clear centre"). Straight (not premultiplied) alpha. Under 8 MiB. Anything else is refused with `[border] <file>: <reason>` and the next choice applies.

**`.ini`.** `key=value` lines like `snes9x-ps5.ini`; unknown keys and `#` lines are ignored; a bad value is ignored with a log line.

| Key | Value | Meaning |
|---|---|---|
| `window` | `x,y,w,h` in surface pixels | Use this rectangle as the window instead of finding it. Must lie inside 1920x1080 and be at least 480x270. The override punches the window out: every pixel of the image inside the rectangle is treated as alpha 0, so the rim is empty (an image's soft glass edge is lost when the window is overridden). |
| `off` | `1` | This game has no border even when Border is Auto. |

**Drop-in.** The Bezel Project's `retroarch/overlay/GameBezels/SNES/*.png` copied to `games/` works as is. Its `.cfg` files are not read.

## Finding the window (B)

1. The pixel at (960, 540) must have alpha below 16, else refuse ("no clear centre").
2. Flood fill (4-connected, explicit stack, no recursion) over pixels with alpha below 16 from there.
3. The window is the bounding box of the filled region. Refuse if it is under 480x270.
4. Measured on five SNES bezels (plan, "What snes9xPS5 does today"): x 261, y 20, w 1398, h 1038 give or take a pixel. A unit test builds a PNG with that window and a soft edge.

## Building the border (B)

From the RGBA image and the window: `backdrop[i]` is the pixel composited on opaque black (`c * a / 255`, alpha forced to 255), so the clear window is black and the rim pixels sit on black: that is what shows in the window outside the picture rectangle (for example when the picture is narrower than the window). `rim` holds every pixel inside the window with alpha 1 to 255 (after a `window=` override has cleared the rectangle, so an overridden window has none) (255 included: an opaque rounded corner must be redrawn over the picture too). `pre` is each colour times alpha, divided by 255 and rounded; `inv` is 255 minus alpha. Allocation failure: log, return null (no border).

## Choosing the border (B)

Border setting 1 (Auto), first hit wins; a candidate that exists but fails to load is logged and skipped:

1. `games/<name>.ini` with `off=1`, for whichever candidate below matched: no border at all.
2. `games/<file_base>.png`, the ROM's own file name without extension (the player's own art always wins, as with covers).
3. The cover matcher (`MatchArtName`, `fe_covers.cpp`; export it through `fe_covers.h` as one wrapper) over the stems of the `games/*.png` files. It already handles the No-Intro name, aliases, the extra-CRC name, spelling, words and the ROM header ([box-art-lookup-plan.md](./box-art-lookup-plan.md)). **Gotcha:** `titles::IndexOf` and `artmatch::IndexOf` cache an index by the address of the name vector and rebuild only when its size changes. The pack list changes between runs, so build the pack's indexes fresh when the folder is read (1500 names is a few milliseconds, once per game load), or key the cache properly; do not hand them a long-lived vector whose contents can change at the same size.
4. `default.png`.
5. The built-in frame (`frontend/assets/borders/default.png`, embedded with `.incbin` the way `fe_text.cpp` embeds fonts, path passed by a `-D` flag in the Makefile).

Border setting 2 (Default frame): steps 4 and 5 only. Setting 0: `SetBorder(nullptr)`.

`LoadGame` knows only the path. Build the `GameInfo` the way `ScanGames` names a game (move that naming into one function in `fe_games.cpp`, behaviour unchanged, existing tests cover it) with the CRC the core computed for the loaded ROM (`Memory.ROMCRC32`; check the name with `rg`). Do not change `PlayGame` or `main-boot.cpp`.

Decode happens on the thread that loads the game, before emulation starts, from `/data` (local). It adds an estimated 30 to 60 ms to a game start; measure on the host and say it is unmeasured on the console. Never load from a network path (README rule 2). PNG decoding uses a file-local `stb_image` copy in `fe_borderimg.cpp` (`STB_IMAGE_IMPLEMENTATION`, `STB_IMAGE_STATIC`, `STBI_ONLY_PNG`, `STBI_NO_STDIO`, `STBI_MAX_DIMENSIONS 8192`), as `fe_covers.cpp` does; do not edit `fe_covers.cpp` for it.

## Renderer (A)

All in `ProsperoVideo.cpp`. `DrawSnes` keeps its signature; it reads the border set by `SetBorder`.

1. **Bounds.** `ComputeRect(src_w, src_h, aspect, bounds)` where `bounds` is the whole surface with no border, else `border.window`. Inside `bounds`, centred, width kept even:
   - 4:3 and 8:7 (the ratio is `base_w / lines` for pixel perfect, as today): fit by the limiting side; if `bounds` has the same ratio within 3%, fill `bounds` instead.
   - Integer: `k = min(bounds.w / base_w, bounds.h / lines)`. If `k < 1` (a window smaller than the source; the 480x270 minimum keeps this from happening for 256x239 and smaller) use the 4:3 rule instead. The result always lies inside `bounds`: clamp, never centre an oversized rectangle (a negative origin would write outside the surface).
   - Stretch: fill `bounds`.
   With the whole surface as bounds every result equals today's (1440x1080, 8:7 at full height, integer, 1920x1080); a host test pins this.
2. **Clear.** Today any change of source size, aspect, scanlines or shader fills the whole surface black and reports the full surface as changed. With a border: restore the backdrop (`memcpy` of 8.3 MB into the surface) and report the full surface only when the picture rectangle changed, the border changed, or `InvalidateSnes()` ran. A change of source size that leaves the rectangle alone rebuilds the lookup tables and nothing else. Without a border keep today's behaviour exactly.
3. **Rim.** After the picture (plain path or `ps5crt::Render`), blend each rim pixel that lies inside the picture rectangle: per colour channel `out = (pic * inv + 127) / 255 + pre` (within 1 of exact). The `Present` rectangle stays the picture rectangle. Pixels of the rim outside the picture rectangle are already in the backdrop.
4. **Log.** Add the border to the existing "picture ..." log line only when one is active (window and picture rectangle).
5. **Threading.** `SetBorder` and `DrawSnes` run on the main loop's thread today; keep the border in a `std::shared_ptr` that `SetBorder` publishes with `std::atomic_store` and `DrawSnes` reads once per call with `std::atomic_load` (the C++17 free functions), so a change of threading cannot race.

## Settings row (C)

A `Border` row in `SettingsMenu` after `Aspect ratio` (enum entry `S_BORDER`, label `Border`, values `Off`, `Auto`, `Default frame`; left/right cycles). It appears in the pause menu for free (the pause menu lists the same rows). `ChangeSetting` already ends in `emu::ApplySettings()`, which calls `Reselect`. Keep the pad hint line as it is. Inserting the enum value changes the pause menu's row numbering (`I_SET0 + s`); check `fe_menu.cpp` for any hard-coded index.

## Default art (C)

`ps5/tools/make_default_border.py` (Pillow, a developer-time tool; the repo does not depend on Pillow to build) draws one 1920x1080 palette PNG: a dark console-style frame with a rounded glass window of about 1398x1038 centred, a soft 6 pixel edge, a thin highlight and a small label area that stays empty. Original work, no third-party pictures or fonts. Deterministic (same bytes every run), under 150 KB, checked in at `ps5/frontend/assets/borders/default.png`; the generator is the source. Document in `ps5/README.md`: what a border is, the folder, naming, the Bezel Project drop-in, the `.ini` keys, and that the default is Off.

## Tests (`make -C ps5 test`)

| Test | Thread | Asserts |
|---|---|---|
| `tests/test_border_render.cpp` | A | `ComputeRect` equals today's numbers for the four aspects, and with a 480x270 window every aspect stays inside it; the 3% fill rule; a border sets the picture rectangle inside the window; the `Present` rectangle for a normal frame is the picture rectangle; a source-size change that keeps the rectangle does not restore the backdrop; rim blend matches the formula; no border leaves the surface as it was. |
| `tests/test_border_pack.cpp` | B | Window found on a generated RGBA, RGB and palette+`tRNS` PNG; refused: wrong size, opaque centre (an RGB image without `window=`), window under 480x270; an RGB image with `window=` leaves the game visible (empty rim); `window=` override; lookup order (own file name, matcher, `default.png`, built-in); `off=1`; a corrupt PNG falls through; rim and backdrop values at known pixels. |
| host render group in `tests/run-tests.sh` | C | With a generated border and `border=1` in the ini, `check_ppm.py` finds the border colour outside the window, the game colour inside it, and (pause menu) both darkened. Border Off: the picture rectangle and colours equal the existing groups'. |

## Left unverified until Jason uses it on the console

How it looks at 1080p and on the 720p fallback; the real cost of the backdrop copy and the rim; memory with a border active; the decode time of a Bezel Project PNG at game start.
