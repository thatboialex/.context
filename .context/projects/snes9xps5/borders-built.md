# Borders: what the renderer and the pack code built

The "renderer" and "packs and lookup" threads' notes. Plan: [borders-plan.md](./borders-plan.md), spec: [borders-spec.md](./borders-spec.md), the Border row and built-in frame: [borders-ui.md](./borders-ui.md). Checked against snes9xPS5 `main` at `d009cb7` (2026-10-17).

Status: merged in order seam #14 (`bb6f39e`), pack and lookup #16 (`c34d035`), renderer #17 (`28654b8`), UI and frame #15 (`d009cb7`). Host-tested; nothing seen on a console.

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

## Not verified

The look at 1080p and on the 720p fallback; the real cost of the backdrop copy and the rim blend; memory with a border active; decode time of a Bezel Project PNG on the console.
