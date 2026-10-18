# Hardware test protocol

How Jason tests a phase on the console. Cloud threads cannot run it. A thread that needs a run writes the exact steps for its phase here under "Runs" and asks Jason once.

A PC run on stock Eden can settle some of the questions below before the console is used; see [pc-test-plan.md](./pc-test-plan.md) for what it can and cannot show.

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

### Questions only the console can settle (from `findings.md`)

Run with the phase it names; each gets one line in that phase's Result.

| Phase | Check | Why |
|---|---|---|
| 1 | The game's version string on screen or in the log (must be 13.0.5) | findings 1: GetDisplayVersion comes from the update's NACP |
| 3 | Log shows "ExeFS: LayeredExeFS patches applied successfully" and the mod's `main.npdm` in `metadata.Print()` | findings 3 |
| 3 | Skyline's first log line in the ProsperoEden log (`Debug_Emulated`); a hook has an effect | findings 4, 6 |
| 3 | Console with no network route: Skyline still starts, no hang | findings 4: log port 6969 |
| 5 | `ninput::init()` outcome: ARCropolis starts, pads work | findings 9 |
| 5 | A mod folder written in capitals and one in lower case | findings 5: case |
| 5 | First start time with a large `ultimate/mods` (hundreds of files) | findings 5: listing and free space |
| 5 | Offline start with `auto_update` on | findings 2 |
| 2 | After a fresh start of the app, `/data/prosperoeden/user/sdmc` does not exist (if it did not before); after starting any game it does, and the log has `[ProsperoEden] sd card: sd:/ is ...` | phase 2: lazy creation |
| 2 | `ultimate/mods` written once as `Ultimate/Mods`, once as `ultimate/mods` | findings 5: case; decides whether guest paths need folding |
| 5 | `time` and `nifm` calls in the log do not fail | findings 4, unverified |
| 7 | Resident memory with Skyline, then with ARCropolis and N mods | findings 8 |
| 7 | Compile ahead on, hooks installed: no crash, no stale code | findings 6 |
