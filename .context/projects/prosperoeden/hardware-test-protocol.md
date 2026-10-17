# Hardware test protocol

How Jason tests a phase on the console. Cloud threads cannot run it. A thread that needs a run writes the exact steps for its phase here under "Runs" and asks Jason once.

Everything below comes from the ProsperoEden source and docs, not from a console run: it is **unverified** until a phase's run closes it. If a step does not match what the console shows, say so in the thread; the thread fixes this file.

## Setup (once, phase 1 completes this)

1. **Build.** The pinned ProsperoEden release ZIP, or the build from the PR (`make install PS5_HOST=<address>` with ProsperoEden closed). A release build is enough.
2. **Game files folder** (chosen in **Settings > Game files**): `keys/` with `prod.keys`, `firmware/` with the extracted `.nca` files, and `roms/Smash/` holding the game and its update (13.0.x). No `mods/` folder in it for a vanilla run.
3. **Settings:** **Diagnostics > Detailed logging** on, **Performance > Memory** 8 GiB. Everything else as you normally use it.
4. **Logs.** ProsperoEden writes `heap.log` and `stderr.log` to `/data/prosperoeden/logs`. The next time ProsperoEden is **opened** they are renamed `*.prev.log` and the older ones are gone, so copy the folder off the console (FTP) **before** reopening ProsperoEden. A crash report is written beside them.

## Every run

1. Close ProsperoEden if it is open, then open it fresh, so the logs hold only this run.
2. Start the game from the Library and do what the phase's row in Runs says.
3. Quit with Touchpad + L1. Wait ten seconds so the logs are written out.
4. If anything went wrong, copy `/data/prosperoeden/logs` (and the crash report) off the console before reopening ProsperoEden, zip it and attach it to the thread, with what you saw: black screen, crash, menu, mods applied or not. On a pass, a line saying so is enough.

## Measuring (phase 7 only, optional before)

Not needed to pass any earlier phase. Phase 7 asks Jason for these when it compares modded Smash with vanilla.

- Load time: a stopwatch from Cross in the Library to the title screen and to character select. Run twice: the first start builds the shader cache, the second is warm.
- Overlay: Touchpad + R1 turns it on and off in a game; it shows renderer, resolution, FPS, worst frame, CPU/GPU load, and `MEM used/pool` with `GFX`.
- Settings to write down: renderer, resolution, refresh rate, Performance switches and speed limit (the `Performance:` and `Memory layout:` lines in `stderr.log` say what the session got).

### Reading the logs

- Frame rate per five-second window: `heap.log` lines starting `EDEN_VULKAN_FRAME`, fields `fps=` and `worst_ms=`. For a match only, take the last N windows (N = match seconds / 5; a 3:00 match is 36, less a couple if Jason quit a few seconds after it ended):

  ```
  grep "^EDEN_VULKAN_FRAME " heap.log | tail -n 34 | sed -E "s/.*fps=([0-9.]+) worst_ms=([0-9.]+).*/\1 \2/" | sort -n | awk '{f[NR]=$1; s+=$1; if($2>w)w=$2} END{printf "windows %d  fps min %.1f  median %.1f  mean %.1f  max %.1f  worst frame %.1f ms\n", NR, f[1], f[int((NR+1)/2)], s/NR, f[NR], w}'
  ```

  Replace `34` with N.
- `tools/analyze-perf-log.py heap.log` (in `thatboialex/ProsperoEden`) needs the `EDEN_DEV_*` lines that only a development build (`make dev`) prints. On a release build it prints "no report window has a previous one to difference against". Use it only if a phase asks Jason for a development build.
- Launch lines in `stderr.log`: `Memory layout:`, `Performance:` (the switches and speed limit the session got) and `Shader cache ready: N pipelines in M ms`.
- Load times are not logged; the stopwatch times are the numbers.

## Runs

Filled in by each phase.

| Phase | Run | Pass means |
|---|---|---|
| 1 | Vanilla Smash 13.0.5, no mods. Start it from the Library, reach the character select screen, play one match (any stage and fighters), quit with Touchpad + L1. Nothing to time or write down | Smash boots to character select and a match plays. If not: the logs folder and crash report, and stop |
| 3 | Smash with the Skyline mod only | Log shows Skyline started; game reaches the title screen |
| 5 | Add ARCropolis and one custom-character ARC mod | Character is selectable and playable |
| 7 | Vanilla, Skyline only, Skyline + ARCropolis with a few and with many ARC mods; measure as in "Measuring" (optional: stopwatch times, overlay MEM/GFX, fps windows) | Numbers recorded in `baseline.md`, modded no worse than the phase allows |
| 8 | Fresh setup by following only the guide | Same result as phase 5, no extra steps |
