# Installing a build and the home-screen icon

Why a second Snes9x PS5 icon can appear, and what the installer does about it. Code: `ps5/coreorbis/orbis-shims/ProsperoInstall.cpp` (install), `ProsperoMigrate.cpp` (finds other copies), `ps5/installer/installer_main.cpp` (the notification). Tests: `ps5/tests/run-tests.sh` groups 8 and 24.

Status 2026-10-17: draft PR in snes9xPS5; host-tested, console unverified. The cause of Jason's second icon is not yet known (see "Open").

## What a send of the ELF does

1. Writes the app folder `/data/homebrew/PPSA99922/` (eboot.bin, sce_module/libc.prx, sce_sys/param.json, icon0.png, pic0.dds, pic1.dds). A file is rewritten only when its bytes differ, so a new build rewrites `eboot.bin` and nothing else.
2. Copies the art to `/user/appmeta/PPSA99922/` when that folder exists.
3. Registers nothing. ShadowMountPlus (SMP) scans `/data/homebrew`, registers the title and puts the icon on the home screen.

## What is the same in every build (checked)

- Title ID `PPSA99922`, content ID `UP9000-PPSA99922_00-SNES9XPS50000001`, `contentVersion` `02.060.000`, install folder: in `main`, in every branch of the repo since 2.6, and inside the CI ELF of 54bf888 (strings of the ELF).
- CI (`.github/workflows/build.yml`) passes no title ID, version or run number; nothing in the install path is random or time-based.
- So a new build of `main` is an update of the same title for SMP. The branches `feature/*`, `fix/*` and `upstream/*` (2.3 to 2.5) still build as `PPSA99009`: a build of one of those is a second icon.

## Ways to get a second icon (any one is enough)

| Cause | How to tell |
|---|---|
| An older build (2.5 or earlier, or a CI build of an old branch) installed `PPSA99009` | `/data/homebrew/PPSA99009` exists; the notification names it |
| An old ELF still in the autoload folder installs it again at every boot | the old folder keeps coming back after deleting it |
| A second folder with our `param.json` (copied folder, another title ID) | the notification names it |
| SMP registers the title again after an update (not known; not in this repo) | only one folder, still two icons: needs the SMP log |

The installer never removes another folder: `PPSA99009` may be another app's.

## Clean-up by hand (once)

Delete the extra folder over FTP, then on the home screen select the extra icon, OPTIONS, Delete (deleting only the icon is not enough: SMP finds the folder again). Games, saves, states and covers are in `/data/snes9x` and stay.

## Open

- Jason's second icon: which folders are in `/data/homebrew` and what `/data/snes9x/logs/installer.log` says after a send (it has one `[migrate]` line per folder).
- Whether SMP registers again after an update of the same title: only the console can say.
