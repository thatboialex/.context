# Hardware test protocol

How Jason tests a phase on the console. Cloud threads cannot run it. A thread that needs a run writes the exact steps for its phase here under "Runs" and asks Jason once.

Everything below comes from the ProsperoEden source and docs, not from a console run: it is **unverified** until phase 1's run closes it. If a step does not match what the console shows, say so in the thread; the thread fixes this file.

## Setup (once, phase 1 completes this)

1. **Build.** The pinned ProsperoEden release ZIP, or the build from the PR (`make install PS5_HOST=<address>` with ProsperoEden closed). A release build is enough; no development build is needed. Open **About** and note the version.
2. **Game files folder** (the folder chosen in **Settings > Game files**):
   - `keys/` with `prod.keys`, `firmware/` with the extracted `.nca` files.
   - `roms/Smash/` holding the game and its update (update version 13.0.x; note the exact version). No `mods/` folder in it for a vanilla run, and no other game needs to be present.
3. **Settings** (the global ones; also open **Library > Game settings** for Smash and check nothing there is changed from Default):
   - **Diagnostics > Detailed logging**: on.
   - **Performance**: every switch at its default (all off except Reactive flushing, which is on), **Speed limit** 100%, **Memory** 8 GiB (the default for these tests; a vanilla run at 4 GiB is optional, see Runs).
   - **Video**: renderer Vulkan (the HUD then starts with `VLK`). Leave resolution, filter and refresh rate at what you normally use, and write down what they are.
   - Frame generation and post-processing off.
4. **Performance overlay.** Touchpad + R1 turns it on and off in a game. It shows, in one line: `VLK 1920x1080 60FPS 18.0MS CPU .. GPU 43% MEM 4.1/8.0G GFX 2.5G` (renderer, the game's resolution, frames per second, the worst frame of the last second in ms, load, memory taken of the pool, and the graphics' part of it).
5. **Logs.** ProsperoEden writes `heap.log` (standard output, including a frame-rate line every five seconds) and `stderr.log` (launch and loader lines) to `/data/prosperoeden/logs`. The next time ProsperoEden is **opened** those two are renamed `heap.prev.log` and `stderr.prev.log` and the one before that is gone. So copy the logs off the console (FTP) after the run and **before** reopening ProsperoEden. A crash report is written beside them.

## Every run

1. Note ProsperoEden version/PR, Smash version, mods present (names only), and the settings above if you changed any.
2. Close ProsperoEden if it is open, then open it fresh, so the logs hold only this run.
3. Turn the overlay on once the game is running (Touchpad + R1).
4. Start Smash from the Library with a stopwatch: press Cross on it and start timing.
5. Note the time to the title screen (press nothing until it shows). Then press through to **Smash** and note the time on its character select screen (the time from Cross to there is the number; menu time after the title screen is yours, so go straight on).
6. Quit with Touchpad + L1. Wait ten seconds so the logs are written out.
7. Over FTP, copy `/data/prosperoeden/logs` (and the crash report, if any) off the console **before opening ProsperoEden again**, zip it and attach it to the thread, with what you saw: black screen, crash, menu, mods applied or not.

## Reading the logs (for the thread that fills `baseline.md`)

- Frame rate per five-second window: `heap.log` lines starting `EDEN_VULKAN_FRAME`, fields `fps=` and `worst_ms=`. For the match only, take the last N windows (N = match seconds / 5; a 3:00 match is 36, less a couple if Jason quit a few seconds after it ended):

  ```
  grep "^EDEN_VULKAN_FRAME " heap.log | tail -n 34 | sed -E "s/.*fps=([0-9.]+) worst_ms=([0-9.]+).*/\1 \2/" | sort -n | awk '{f[NR]=$1; s+=$1; if($2>w)w=$2} END{printf "windows %d  fps min %.1f  median %.1f  mean %.1f  max %.1f  worst frame %.1f ms\n", NR, f[1], f[int((NR+1)/2)], s/NR, f[NR], w}'
  ```

  Replace `34` with N.
- `tools/analyze-perf-log.py heap.log` (in `thatboialex/ProsperoEden`) needs the `EDEN_DEV_*` lines that only a development build (`make dev`) prints. On a release build it prints "no report window has a previous one to difference against". Do not use it for the baseline; use it only if a later phase asks Jason for a development build.
- Launch lines in `stderr.log`: `Memory layout:`, `Performance:` (the switches and speed limit the session got) and `Shader cache ready: N pipelines in M ms`.
- Load times are not logged; the stopwatch times from Jason are the numbers.

## Runs

Filled in by each phase.

| Phase | Run | Pass means |
|---|---|---|
| 1 | Vanilla Smash, no mods, 8 GiB. **Run A** first start: stopwatch from Cross to title and to character select (the shader cache is cold, expect it to be slow). Quit, reopen ProsperoEden, **run B** the same start again (warm cache) to get load times. In run B, after character select, play one match: Final Destination, 3:00 time, no items, you as Mario against a level 5 CPU Mario; note the overlay's MEM and GFX at the title, at character select and at the end of the match. Quit with Touchpad + L1 from the results screen. Optional **run C**: the same at Memory 4 GiB, to see whether Smash needs the larger layout | Baseline load times and match frame rate recorded in `baseline.md` |
| 3 | Smash with the Skyline mod only | Log shows Skyline started; game reaches the title screen |
| 5 | Add ARCropolis and one custom-character ARC mod | Character is selectable and playable |
| 7 | Same, with the performance settings the phase changes | Frame rate and load time compared with `baseline.md` |
| 8 | Fresh setup by following only the guide | Same result as phase 5, no extra steps |
