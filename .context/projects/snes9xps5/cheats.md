# Cheats: loading, saving and live reload

How cheats get into a running game in `thatboialex/snes9xPS5`. Code: `ps5/frontend/fe_emu.cpp` (`LoadManualCheatsForRom`, `SyncCheatsFromFile`, `SaveCheats`), the list is `CheatMenu()` in `fe_menu.cpp`. Test: `ps5/tests/run-tests.sh` group 32.

Status 2026-10-17: live reload is a draft PR in snes9xPS5; host-tested (group 32), console unverified.

## How it works

1. **Where the codes live.** `/data/snes9x/cheats/<ROM file name>.cht`. Without that file, the folder (and its `database/` and `Nintendo - Super Nintendo Entertainment System/` subfolders, and the ROM's own folder) is searched for the `.cht` whose name matches the ROM or its official title (`fe_cheatlookup.h`). Both Snes9x's own format and RetroArch's `cheatN_desc/code/enable` format load.
2. **Toggles are saved at once** to `<ROM file name>.cht` in Snes9x's format, so after the first toggle that file is the one that counts, even when the codes came from a RetroArch file.
3. **Live reload (new).** The app remembers the file the cheats came from (path, size, time: `State::cheat_file`) and looks at it again:
   - while a game runs, once a second (`RunFrame`): a `stat` of `<ROM>.cht`, or of the file the cheats came from when `<ROM>.cht` does not exist. No folder search;
   - when the cheat menu opens (at once) and while it is open (once a second). With no cheats loaded the menu also searches the folder, as before, so a file with a different name is found there;
   - before a toggle or All ON/OFF, and before the file is written at the end of a game.
4. **A changed file replaces all cheats in the game** (`S9xDeleteCheats` gives the game its bytes back, then the file is loaded, and the file's `enable` flags are the state). A reload shows "Cheats updated: N" in the picture. A file seen to change is loaded only when it looks the same on the next look, so an upload that is still running is not read half done.
5. **The file wins.** Toggles are saved on every press, so the file and the game agree except when a write failed. If the file changed since it was read, a toggle reloads the list instead of toggling (the list under your finger is new) and the end-of-game save does not overwrite it.
6. **A file with no usable cheat** (half a copy, emptied, wrong format) changes nothing: the cheats that run stay (rolled back from a copy), the log says so, and the file is not looked at again until it changes. At the end of the game the running cheats are written back over it.

## Limits

- Snes9x applies cheats to RAM and battery RAM only (`S9xSetByteFree` ignores ROM). ROM-address codes do nothing, as before.
- A file with another name than `<ROM>.cht` is found only while the cheat menu is open and the game has no cheats (the folder search is a directory scan, kept off the emulation thread's per-second look).
- Replacing a file with the same size inside the same second as the last look is not seen (the look compares size and whole seconds of the time, as `fe_coverworker.cpp` does).
- The per-second `stat` runs on the emulation thread, like the state and battery saves do; it is a local file under `/data`, never a share. The hard rule on waits (README, rule 2) is about network and lock waits.
- Console unverified: only the host build has run it. A code added over FTP to a game being played is the check to make.
