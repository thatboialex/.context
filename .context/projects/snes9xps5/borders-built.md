# Borders: what the renderer and the pack code built

The "renderer" and "packs and lookup" threads' notes. Plan: [borders-plan.md](./borders-plan.md), spec: [borders-spec.md](./borders-spec.md), the Border row and built-in frame: [borders-ui.md](./borders-ui.md). Checked against snes9xPS5 `main` at `d009cb7` (2026-10-17); the section "Aspect ratio: the window follows the picture" at `a4c069a` (#20); "Integer scale" at #22.

Status: merged in order seam #14 (`bb6f39e`), pack and lookup #16 (`c34d035`), renderer #17 (`28654b8`), UI and frame #15 (`d009cb7`), window follows the Aspect ratio setting #20 (`a4c069a`). Host-tested; nothing seen on a console.

## Renderer (`ps5/coreorbis/orbis-shims/ProsperoVideo.h/.cpp`)

- `ps5video::Border` (backdrop, window, rim) and `SetBorder` are as the spec's "Seams". The border is published and read with `std::atomic_store` / `std::atomic_load`.
- `ComputeRect(src_w, src_h, aspect, bounds)` is public; with the whole surface as bounds it gives today's numbers. The picture rectangle is clamped inside `bounds`.
- The backdrop is copied only when the picture rectangle or the border changes, or after `InvalidateSnes()`; a source-size change that keeps the rectangle rebuilds the lookup tables only.
- Test: `make -C ps5 test-border-render` (part of `make test`); it has its own block in `ps5/Makefile`.

## Pack and lookup (`ps5/frontend/fe_border.*`, `fe_borderimg.*`)

- `fe::border::Select(rom_path, rom_crc)`, `Reselect()` and `Release()` are called from `emu::LoadGame`, `emu::ApplySettings` and `emu::CloseGame`. `Reselect` does nothing while the applied setting is unchanged (every settings change passes through it).
- The game is named as the scan names it: `DescribeGame(path, crc)` in `fe_games.cpp`, shared with `ScanGames`. The pack's names are matched with `MatchPackName` in `fe_covers.cpp`, which builds fresh matcher indexes (the built-in lists' cache is keyed by vector address and would go stale).
- Beyond the spec: a border with over 400000 rim pixels (`kMaxRim`) is refused, logged, and the next choice applies; `games/<ROM name>.ini` with `off=1` works even with no picture of that name.
- Test: `make -C ps5 test-border` and host test group 34. Host decode, window search and build took 14 ms at -O2 (reported by the thread); not measured on a console.

## Aspect ratio: the window follows the picture (snes9xPS5#20, `a4c069a`, 2026-10-17)

Jason: the border "only works in 4:3". Cause: the window was whatever the art drew (4:3 for The Bezel Project and the built-in frame), and the Aspect ratio setting only placed the picture inside it, so 8:7 left black bars in the glass and Stretch 16:9 filled a 4:3 hole.

**Default picked: the border's window takes the shape of the Aspect ratio setting, by stretching the art** (`FitOpening`, `RefitImage`, `FitToAspect` in `ps5/frontend/fe_borderimg.*`; called from `Build` in `fe_border.cpp`, so Choose, Select and the download watcher all get it).

| Aspect ratio | Window |
|---|---|
| 4:3 | as drawn when within 3% of 4:3 (the same snap as the renderer's `FitRatio`); else the height is kept and the width follows |
| 8:7 (pixel perfect) | height kept, width = height x 8/7 (1186 for the 1398x1038 art); the side margins stretch wider |
| Stretch 16:9 | the side margins may shrink to 40% (`kMinMarginPct`), which caps the width (1712 for 1398x1038); the height follows the shape (963); top and bottom margins stretch taller |
| Integer scale | the window is cut to the whole-multiple picture, 4x = 1024x896 (snes9xPS5#22, see below); #20 first left it as drawn, with black around the picture |

- The margins keep their drawn proportions (left : right, top : bottom), so an off-centre frame stays off-centre. A window that touches the surface edge (no margin) is centred when it narrows.
- The art is cut in five pieces per axis: margin, 64 px of window corner, window middle, corner, margin. Only margins and the middle stretch; the corners (rounded glass, soft edge, opaque corner pixels) are copied 1:1 and nothing is blended across a cut. Bilinear, colours weighted by alpha, fast path for opaque pixels. About 40 ms on the host at -O2 for half-transparent art (typical art is mostly opaque); it runs where the border is picked (a settings change, a game start, a download landing), never on the emulation thread.
- Not stretched to match: pixel perfect on a 239-line PAL picture (256:239 is 6% off 8:7) keeps small pillar bars inside the 8:7 window; the renderer still fits it by its limiting side. Hi-res and interlaced pictures are the same shape as their base picture.
- `Reselect()` now also fires when the aspect ratio changed (Border not Off), so the pause menu's Aspect ratio row refits at once. `Choose` takes the aspect (default Integer = as drawn, which is what older tests compare). The log line says which aspect the window is for.
- Tests: `make -C ps5 test-border` (TestFit, TestAspect: window shapes, corners as drawn, no blend across cuts, the window search finds the new hole, Reselect on aspect change, a downloaded border arriving fitted); run-tests.sh group 35 f (aspect=3 and aspect=1 from the ini) and g (pause menu Aspect ratio row).
- Open: how the stretched frame looks on a TV (bezel art with box art or text at the sides gets 1.4x wider at 8:7 and 0.4x at 16:9) is unverified on a console. If Jason prefers a plain "hide the border for non-4:3", the switch is `OpeningRatio` returning false for the aspect.

## Integer scale: the window is the picture (snes9xPS5#22, 2026-10-17)

Jason (photo, Super Robot Wars, Japan border): the game sat in the middle of the opening with black bands. Cause: the picture was 73% of the window's width, exactly 4x (1024x896) in the 1398x1038 window; #20 deliberately left Integer alone (`OpeningRatio` false), so only 4:3, 8:7 and Stretch fitted.

**Default picked: Integer cuts the window to the whole multiple itself** (`IntegerOpening` in `fe_borderimg.*`): the largest k with 256k x 224k inside the drawn window (4 for any window of at least 1024x896), margins keep their proportions as in #20, so 1398x1038 at 261,21 becomes 1024x896 at 448,92. The art around it is stretched to fit (top and bottom about 4.4x taller, sides 1.7x wider for the Bezel Project's usual art); it looks like the 8:7 refit, only more so. The picture is smaller than at 4:3 (1024x896 against 1398x1038), which is what Integer scale means.

- A 239-line (PAL) picture has no whole multiple in a 1024x896 window (4x is 956 high). `ComputeRect` takes a window that is itself a whole multiple of 256x224 and a picture of another height as "fill it like 8:7": 960x896 with 32 px bars each side, not 3x with black all round. On the bare screen, Integer is still the whole multiple (1024x956).
- `Aspect::Count` is now the "art as drawn" value (tests; `Choose` default). The settings screen never gives it.
- Tests: `make -C ps5 test-border` (TestFit: `IntegerOpening`, art, TestAspect), `test-border-render` (`ComputeRect` in a 4x window, 239 lines), run-tests group 34 (the logged windows are the fitted ones now) and 35 f (aspect=2: 448,92 to 1472,988, the picture fills it). Group 35 has no 239-line case: the test ROM draws 224 lines.
- Open: unverified on a TV. If the stretch looks bad, the alternative is to treat Integer as 8:7 when a border is on (non-integer picture filling the window).

## Not verified

The look at 1080p and on the 720p fallback; the real cost of the backdrop copy and the rim blend; memory with a border active; decode time of a Bezel Project PNG on the console.
