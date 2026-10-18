# PC test plan: Skyline and ARCropolis on Eden, then `make test`

Status: Proposed, 2026-10-18. Requested by Jason. For an AI coding agent (Codex or Claude Code) running on Jason's own PC. It can be followed from this file alone; the facts it relies on are in [findings.md](./findings.md).

## Purpose

Find out, before the next console run, whether Skyline and ARCropolis 4.1.1 start Super Smash Bros. Ultimate 13.0.5 and load a custom character in **upstream Eden at the version ProsperoEden pins**, on a PC. Eden is the emulator inside ProsperoEden, so a failure here is a failure on the console too, and a pass removes the emulator as the suspect. Also run `make test`, which cloud threads cannot ([findings.md](./findings.md) section 7).

## What a PC run cannot show

Do not claim any of these from a PC run; they stay **unverified** until Jason's console run ([hardware-test-protocol.md](./hardware-test-protocol.md)).

| Not covered | Why |
|---|---|
| PS5 memory (4/6/8 GiB layout, findings 8) | PC has far more memory and a different allocator. |
| ARCropolis's load-address check (findings 9) | Stock Eden on PC picks its own ASLR offset; ProsperoEden's `load_address.h` is not in play. A PC result for `ninput::init()` is a hint, not an answer. |
| PS5 graphics, shader cache, frame rate, load times | Different GPU path (RADV on the PS5). |
| ProsperoEden's Eden patches (`tools/eden-patches/01` to `04`, the `eden_patch` string replacements in `headless/CMakeLists.txt`) | This plan builds stock Eden. Mods and SD card paths differ from ProsperoEden's `roms/<Game>/mods/` and `user/sdmc`. |
| Launcher screens (Library, `Skyline, 12 ARC mods`, phase 4) | PS5-only UI. |
| Case sensitivity, big-directory listing and free-space behaviour of the PS5 file system (findings 5) | Depends on the host file system. Windows is case-insensitive, so use Linux if you test case. |
| Sockets / `nifm` with no network route (findings 4) | Different network stack. |

## Rules for the agent

1. **Never download, fetch or create** games, keys, firmware, Skyline, ARCropolis or ARC mods ([README.md](./README.md) rule 1, [ADR 001](./decisions/001-user-supplied-skyline.md)). Jason puts them on the PC. If a step needs one that is missing, stop and ask Jason for its path. Do not search the web for them.
2. Do not guess what Skyline or ARCropolis do. If a log contradicts [findings.md](./findings.md), say so and write the contradiction into your report; do not patch around it.
3. Do not edit Eden or ProsperoEden source to make a step pass. This plan is read, build, run, report. A needed fix goes in the report as a proposal.
4. Say plainly what you did not run. "Eden built and Smash reached the title screen; ARCropolis not tested because no ARC mod was supplied" is a complete report.
5. Work in a scratch folder (below). Do not put anything of Nintendo's, Skyline's or ARCropolis's into any git repository.

## Inputs Jason supplies (ask once, record the paths)

| Input | Notes |
|---|---|
| Smash base game | A dump Jason owns (`01006A800016E000`). |
| Smash update **13.0.5** | The only version ARCropolis 4.1.1 supports (findings 1). Update title ID `01006A800016E800`. |
| `prod.keys`, `title.keys` and firmware | Jason's own, from his console. |
| Skyline | A release folder with `exefs/subsdk9` and `exefs/main.npdm` (findings 2). |
| ARCropolis 4.1.1 | `libarcropolis.nro`. |
| One custom-character ARC mod | A folder that ARCropolis reads from `ultimate/mods/`. Ask Jason which character it adds and how it is selected (a slot, a Skin, a new fighter). |
| A PC with a GPU that runs Vulkan, 16 GiB RAM, about 40 GiB free disk | Windows or Linux. Prefer Linux (or WSL for the build only; running a game needs the GPU). |

## Layout used below

`$SCRATCH` is a new empty folder, for example `~/pc-test`. Eden's user directory is `$EDEN_USER`.

```text
$SCRATCH/
  eden-src/        Eden at the pinned commit
  eden-build/      its build output
  ProsperoEden/    thatboialex/ProsperoEden, for make test
  report.md        what you write at the end
```

`$EDEN_USER` is where Eden keeps `keys/`, `nand/`, `sdmc/`, `load/`, `log/`. On Linux it is normally `~/.local/share/eden`; a `user/` folder next to the Eden executable switches Eden to portable mode and is the easier choice for a test (check Eden's build documentation at the pinned commit; this is **unverified** here). Use portable mode so the test cannot touch Jason's other Eden setup.

## Step 1. Get the same Eden as ProsperoEden

1. Read `UPSTREAM.json` in `thatboialex/ProsperoEden` (`main`). Its `eden_commit` is the pin. At the time of writing it is `67bada77f8a43a90da2e94e89b8e7da73c256989`. Use whatever the file says now.
2. Fetch that commit's source from the mirror the repo itself uses (`tools/deps.json`, entry `eden`): `https://github.com/eden-emulator/mirror/archive/<commit>.tar.gz`, or `git fetch` of that commit. Extract to `$SCRATCH/eden-src`. Check the hash in `UPSTREAM.json` `archives` if you used the tarball.
3. Build the **desktop** target by following `docs/Build.md` in that source (it states the compiler, Qt and dependency list for the platform). Release build, tests off. On Linux the usual shape is:

```bash
cmake -S eden-src -B eden-build -DCMAKE_BUILD_TYPE=Release -DYUZU_TESTS=OFF
cmake --build eden-build -j"$(nproc)"
```

   If the build document and these flags disagree, the build document wins. If the pinned commit does not build on the PC's toolchain, report the exact error and stop; do not move to a different Eden commit, because the test is only meaningful at the pin.
4. Record in the report: commit, OS, compiler, and the executable's path.

## Step 2. Vanilla Smash first (must pass before adding anything)

1. Create portable mode (a `user/` folder next to the executable). Put Jason's `prod.keys`/`title.keys` in `$EDEN_USER/keys/`.
2. Install firmware and the Smash base game and the **13.0.5** update through Eden (File > Install Files to NAND, or the equivalent in that build).
3. Start Smash with no mods. Reach the character select screen and start a match.
4. Record: Eden's displayed game version (must read 13.0.5), first-frame time, any crash, and the log file path (`$EDEN_USER/log/`).

If vanilla Smash does not boot, stop here. Skyline cannot be blamed, and this tells Jason the baseline problem is the emulator or the game files, not the mods.

## Step 3. Skyline only

1. Create the mod folder: `$EDEN_USER/load/01006A800016E000/skyline/`.
   - `exefs/subsdk9` and `exefs/main.npdm` from the Skyline release (findings 2).
   - Do **not** add ARCropolis yet. `romfs/skyline/plugins/` stays empty (create it).
2. Start Smash. Pass means: the game reaches the title screen, and Eden's log (set log filter to include `Debug_Emulated` at Info; findings 4) shows Skyline's lines (it logs through `OutputDebugString`).
3. Record the log lines that name the step reached. If it fails, the last Skyline line and the first Eden error after it are the finding. Also note whether a log mentions the `main.npdm` being applied (findings 3).

## Step 4. ARCropolis and one custom character

1. Put `libarcropolis.nro` at `$EDEN_USER/load/01006A800016E000/skyline/romfs/skyline/plugins/libarcropolis.nro`.
2. Put the ARC mod at `$EDEN_USER/sdmc/ultimate/mods/<mod name>/` (findings 2, section 5). This is the SD card folder Eden shows to the game as `sd:/`.
3. Start Smash. Pass means: the title screen, then the custom character is selectable on the character select screen and playable in a match.
4. After the first start, check `$EDEN_USER/sdmc/ultimate/arcropolis/` exists with `logs/`, `config/`, `cache/` (created by ARCropolis, findings 2). Read ARCropolis's newest log in `logs/`.
5. Record the answers PC can give (each is one line in the report; label them *PC, not console*):
   - Did `ninput::init()` start without a hang or crash (findings 9)? Do the pads work in menus?
   - Did `time`/`nifm` calls fail anywhere in either log (findings 4)?
   - Does ARCropolis report "emulator" or "console" for the main module's address, and what address did Eden give (the Eden log prints the load address of `main`)?
6. If the mod does not appear: say whether ARCropolis's log shows the mod folder was discovered, whether it reports a conflict (`conflicts.json`), and whether the character's files were requested. That separates "mod not found" from "mod found, game never asked for it".

## Step 5. `make test` for ProsperoEden

Cloud threads cannot run this because they cannot download dependencies. Run it on Linux (Ubuntu 26.04 or WSL, per `docs/BUILDING.md`):

```bash
git clone https://github.com/thatboialex/ProsperoEden "$SCRATCH/ProsperoEden"
cd "$SCRATCH/ProsperoEden"
git checkout <branch>      # main, or the branch Jason names (phase 2: PR #6, phase 4: PR #7)
make toolchain             # reports missing host tools
make test
```

Record the exit code and the last 40 lines of output. For a failure, record the first failing check by name (`headless/*_check.cpp` or `tools/check-*.py`), not the whole log. If `make toolchain` reports missing tools, list them and stop; installing system packages is Jason's decision unless he says otherwise. The branches to run, newest first: ProsperoEden #7 (phase 4, stacked on #6), then #6 (phase 2), then `main`. Report which of them pass.

## Step 6. Report

Write `$SCRATCH/report.md` and give it to Jason (paste, or commit to `thatboialex/.context` on a branch he names, under `.context/projects/prosperoeden/pc-runs/<date>.md`; no game, key or mod content in it). Use this shape:

```markdown
# PC run <date>
Eden commit: <sha>   OS: <os>   Build: <ok | error + first error line>
Inputs: Smash <version seen>, Skyline <release>, ARCropolis <version>, ARC mod <name>
| Step | Result | Evidence (log line or screenshot name) |
|---|---|---|
| 2 Vanilla boots | pass / fail | |
| 3 Skyline only | pass / fail | |
| 4 ARCropolis + character | pass / fail | |
| 5 make test (main / #6 / #7) | pass / fail | |
PC answers (not console): ninput::init, time/nifm, load address, ARCropolis log summary
Not run: <list>
Proposed fixes: <list, none applied>
```

## What each result means for the plan

| PC result | Consequence |
|---|---|
| Step 2 fails | The baseline is broken before mods: fix the game files, keys or firmware first. Phase 1 on the console will probably fail the same way. |
| Step 3 fails | Skyline does not start on stock Eden: that is the phase 3 work (Opus), now with a log. |
| Step 3 passes, step 4 fails | ARCropolis's problem, not Skyline's: phase 5 starts from the ARCropolis log. |
| Steps 2 to 4 pass | Eden is not the obstacle. Remaining risks are PS5-only: memory, load address, renderer. Jason's console run (phases 3 and 5) becomes a confirmation. |
| Step 5 fails | The failing check names what a code phase broke or what the host lacks. |

## Notes for the next session

- Eden's user-directory paths, the portable-mode folder and the log filter syntax are from the yuzu lineage and Jason's description, **not verified against the pinned source**: the agent should read Eden's own docs at the pin and correct this file where it differs.
- Add the PC run result to the phase file it affects (3 or 5) as a line in its Result block, labelled *PC, stock Eden*.
