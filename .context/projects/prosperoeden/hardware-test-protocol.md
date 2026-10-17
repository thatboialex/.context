# Hardware test protocol

How Jason tests a phase on the console. Cloud threads cannot run it. A thread that needs a run writes the exact steps for its phase here under "Runs" and asks Jason once.

## Setup (once, phase 1 completes this)

1. Console on the pinned ProsperoEden release or the build from the PR (`make install PS5_HOST=<address>`, or the release ZIP).
2. In the game files folder: `keys/`, `firmware/`, and `roms/Smash/` holding the game and its update.
3. **Settings > Diagnostics > Detailed logging** on.
4. Game settings: Memory as the phase says (default for these tests: 8Gb).

## Every run

1. Note ProsperoEden version/PR, Smash version, mods present (names only).
2. Start Smash from the Library. Reach the character select screen. Note how long it took.
3. Quit with Touchpad + L1.
4. Zip `/data/prosperoeden/logs` (and the crash report if any) and attach it to the thread, with what you saw: black screen, crash, menu, mods applied or not.

## Runs

Filled in by each phase.

| Phase | Run | Pass means |
|---|---|---|
| 1 | Vanilla Smash, no mods: reach character select, play one match, keep the performance log | Baseline frame rate and load time recorded in `baseline.md` |
| 3 | Smash with the Skyline mod only | Log shows Skyline started; game reaches the title screen |
| 5 | Add ARCropolis and one custom-character ARC mod | Character is selectable and playable |
| 7 | Same, with the performance settings the phase changes | Frame rate and load time compared with `baseline.md` |
| 8 | Fresh setup by following only the guide | Same result as phase 5, no extra steps |
