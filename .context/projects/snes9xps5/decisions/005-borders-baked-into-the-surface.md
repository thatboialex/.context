# ADR-005: Borders are painted once into the surface, and the picture goes in the art's window

## Status
Proposed, 2026-10-17. Nothing is built. Becomes Accepted when the renderer change is on `main` and Jason has seen a border on the console. Unverified on hardware: the look at 1080p, the cost of the backdrop copy and the rim blend, memory.

## Context

Jason asked for what SwanStationPS5 does: art around the picture while a game plays. Plan: [borders-plan.md](../borders-plan.md).

How SwanStationPS5 does it (design only; the code is GPL-3.0-or-later and may not be copied into our MIT `ps5/`): the picture is drawn at 4:3, then the border PNG (1920x1080, alpha, a clear window) is drawn as a texture over the whole screen on a GPU path, every frame.

How snes9xPS5 draws: the CPU writes the picture into a linear 1920x1080 surface, and `Present` swizzles only the rectangle that changed into the scan-out buffer. After the first frame nothing outside the picture rectangle is written or sent (`ProsperoVideo.cpp`: `DrawSnes`, `TileBlit`). There is no GPU path to put a texture on.

The question: where does the border enter that pipeline so that it costs the emulation nothing and still blends with the picture's edge?

## Decision

1. **The border is composited once into a backdrop and copied into the surface at the moment the surface is cleared to black today.** From then on only the picture rectangle changes, so the border costs nothing per frame and is not sent to the scan-out buffer again.
2. **The picture is drawn inside the border's window**, using the existing Aspect setting, not full screen with the border on top. The window is found from the PNG's alpha (flood fill from the centre; bounding box), with an `.ini` override.
3. **The soft edge is a rim list.** Pixels of the border inside the window with alpha above 0 are kept as a short list and blended over the picture after it is drawn, every frame. It is tens of thousands of pixels, not two million.
4. **Art is plain files in `/data/snes9x/borders/`**, 1920x1080 PNG with alpha, named like the Bezel Project's files so its pack drops in. Per-game art is found with the cover matcher. A built-in original frame is the fallback. No third-party art is bundled and nothing is downloaded in the first release.
5. **Off by default** until Jason has used it on the console.

## Alternatives considered

| Option | Why not |
|---|---|
| Draw the border over the finished frame every frame (SwanStationPS5's way) | Blends 2 million pixels per frame on the CPU, and `Present` would have to swizzle the whole surface every frame instead of the picture rectangle. That adds the cost this design avoids, on every game, for art that does not change. |
| An opaque border only, no rim list | Simplest, but the Bezel Project art has a soft, semi-transparent glass edge and rounded corners inside the window; without the rim they show as a hard cut. The rim costs a few thousand blends per frame. |
| Second scan-out plane or GPU overlay for the border | The app has no GPU driver (`ProsperoCrt.h`); nothing to build on. |
| Draw the picture full height and ignore the art's window | The art assumes its hole; the picture would be covered or show black slivers. |
| Convert art to our own format (`.bmp`, raw RGBA, a manifest) | Players have PNGs from the Bezel Project; a converter is more work for them and for us. `stb_image` is already in the build. |
| Bundle or download the Bezel Project SNES set | The repository states no licence. SwanStationPS5 ships only a name list and downloads on demand; a download here needs a helper-side queue, which is a later phase and Jason's call. |
| On by default | The window is 3% smaller each way than the 1440x1080 picture and no console has shown it yet (README rule 8). |

## Consequences

### Positive
- No cost per frame outside the rim; shaders and aspect settings need no change beyond the rectangle they draw into.
- The pause menu, cheat screen and every other path that already invalidates the picture restore the border by themselves.
- One art format, compatible with an existing 1445-game SNES pack, and the same name matching as covers.

### Negative
- About 9 MB of memory while a border is active; if it cannot be allocated the border is off.
- A game that switches between 256 and 512 pixel modes must not restore the backdrop on every switch (the renderer restores it only when the picture rectangle changes). If this is missed, each switch costs an estimated 1 to 2 ms plus a full-surface present.
- Art that needs to be drawn over the picture other than at its edge (a full-screen scanline or glare layer) is not supported in this design. It would need a per-frame blend of the whole window.

### Neutral
- One more setting key (`border`) and one more folder under `/data/snes9x`.
- The art has to be 1920x1080; other sizes are refused rather than scaled.
