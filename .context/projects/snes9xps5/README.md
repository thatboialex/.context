# snes9xPS5: project context

Entry point for any thread working on `thatboialex/snes9xPS5`. Read this file first, then load only the file your task needs.

## What it is

Snes9x 1.63 as a native PS5 home-screen app (title ID PPSA99922) for jailbroken consoles. It follows PS5SX2's layout. Everything outside `ps5/` is the upstream Snes9x core, unchanged. All new PS5 code lives in `ps5/` under MIT.

## Repos

| Repo | Role |
|---|---|
| `thatboialex/snes9xPS5` | Code. Fork of MisterTemaki/snes9xPS5. Default branch `main`. |
| `thatboialex/.context` | Context, plans and ADRs. Plans and notes go here, not in the code repo. |

## Hard rules

1. Do not edit files outside `ps5/`. Hook the core from `ps5/` (linker `--wrap`, frontend code).
2. No network, disk or lock wait on the emulation, video or audio threads. Waits belong to worker threads. One documented exception: a miss inside an oversize MSU-1 `.msu` data file waits, because skipping data corrupts the game ([spec](./network-shares-spec.md), "Reads" item 6).
3. Saves, states, cheats, covers and settings stay on the console under `/data/snes9x`. Network shares are read-only.
4. Never log or display a password. Log share addresses with credentials removed.
5. Match the neighbouring file's style (tabs and Allman braces in `fe_games.cpp`). New files start with `// SPDX-License-Identifier: MIT`.
6. Do not copy GPL code into `ps5/` MIT files. Reimplement from design notes ([ADR 001](./decisions/001-network-rom-loading.md)).
7. Every change passes `make -C ps5 test` (Linux, ASan/UBSan) and builds with `make -C ps5 ps5`.
8. A cloud session cannot run the console. Label console behaviour "unverified" until Jason confirms it on hardware.

## Terms (use exactly these)

| Term | Meaning |
|---|---|
| **Share** | One configured network location (`smb://` or `nfs://` address) that holds ROMs. |
| **Network ROM** | A ROM that lives on a share. |
| **netfs** | The planned layer that makes shares look like folders to the whole process. |
| **Backend** | One protocol implementation under netfs: SMB (libsmb2) or NFS (libnfs). |
| **Mount point** | The virtual folder a share appears under: `/smb/<host>/<share>/<path>` or `/nfs/<host>/<path>`. |
| **Whole-file fetch** | Reading a complete file into memory when it is opened. |
| **Read-ahead cache** | Block cache plus a worker thread that reads past the reader. Used for big files only. |
| **Shelf** | The 3D game list (`fe_shelf.cpp`). Not "library" or "grid". |
| **Tab** | One of the shelf's views: Internal, USB, Network (`fe_tabs.h`). Not "section" or "page". |
| **Border** | Art around the SNES picture while a game runs (planned). Not "bezel", "overlay" or "frame"; "bezel" only when naming The Bezel Project. |
| **Border pack** | The folder `/data/snes9x/borders/` and the PNG files in it (planned). |
| **Window** | The clear hole of a border; the picture is drawn inside it (planned). |
| **Rim** | Border pixels inside the window with some alpha, blended over the picture each frame (planned). |

## Files

| File | Load when |
|---|---|
| [architecture.md](./architecture.md) | You touch ROM discovery, loading, saves, the build or the tests. |
| [swanstation-nfs-reference.md](./swanstation-nfs-reference.md) | You implement or review anything copied in spirit from SwanStationPS5. |
| [network-shares-plan.md](./network-shares-plan.md) | You work on SMB/NFS loading: goal, design, phases, open decisions. |
| [network-shares-spec.md](./network-shares-spec.md) | You write the code: address format, per-file changes, tests, risks. |
| [netfs-foundation.md](./netfs-foundation.md) | You touch `ps5/netfs/`: what exists, the choices made, what is unverified. |
| [nfs-backend.md](./nfs-backend.md) | You touch `ps5/netfs/backend_nfs.cpp` or the vendored libnfs. |
| [smb-backend.md](./smb-backend.md) | You touch the SMB backend (libsmb2) or debug an SMB share. |
| [decisions/001-network-rom-loading.md](./decisions/001-network-rom-loading.md) | You question why netfs is built the way it is. |
| [box-art-lookup-plan.md](./box-art-lookup-plan.md) | You work on cover lookup for renamed or fan-translated ROMs: goal, design, acceptance. |
| [decisions/002-box-art-name-matching.md](./decisions/002-box-art-name-matching.md) | You question why art names are matched by words and the ROM header. |
| [decisions/003-network-shares-screen.md](./decisions/003-network-shares-screen.md) | You touch the Network shares screen, the keyboard or how `shares.txt` is edited. |
| [decisions/004-shelf-tabs-network-tab.md](./decisions/004-shelf-tabs-network-tab.md) | You touch the shelf's tabs, the Network tab's states or its Square scan; what was and was not taken from PSFlycast. |
| [cheats.md](./cheats.md) | You touch cheat loading, saving or the cheat menu: where `.cht` files are found, and how a changed file reaches the running game. |
| [install-and-icons.md](./install-and-icons.md) | You touch the installer or `ProsperoMigrate`, or Jason sees a second Snes9x icon: what a send writes, what stays fixed, what makes a second icon. |
| [borders-plan.md](./borders-plan.md) | You work on borders around the picture: goal, what SwanStationPS5 does, defaults, the three-thread split, acceptance. |
| [borders-spec.md](./borders-spec.md) | You write the border code: interfaces between the threads, pack format, window detection, renderer changes, tests. |
| [decisions/005-borders-baked-into-the-surface.md](./decisions/005-borders-baked-into-the-surface.md) | You question why the border is painted once into the surface and the picture goes in the art's window. |

## Status

| Date | State |
|---|---|
| 2026-10-17 | Network shares: plan approved and being built as draft PRs in snes9xPS5 (shared layer #7, NFS backend #6). Phase 0 spikes still need a console. |
| 2026-10-17 | SMB backend written, host-tested against Samba and built in CI (snes9xPS5#8); not run on a console. |
| 2026-10-17 | Box art lookup for translated ROMs: implemented in a draft PR (words, extra CRC table, ROM header); host tests pass, console unverified. Covers for games libretro lacks now come from an embedded Hasheous/TheGamesDB table (step 7, 411 games), and `Front Mission - Gun Hazard` no longer gets `Front Mission`'s cover; a live per-ROM lookup stays open decision D4 and is not built. |
| 2026-10-17 | Network shares screen (Settings > Network shares, with an on-screen keyboard, connection test, edit and remove) built on the netfs core; draft PR in snes9xPS5, unverified on console. |
| 2026-10-17 | Network shares: the shared layer, cache, app wiring (CRC naming, late covers), NFS and SMB backends and the shares screen are on main (snes9xPS5 #7, #6, #8, #10), CI green. Nothing has run on a console yet; `.msu1` packs on a share are not prefetched. |
| 2026-10-17 | Shelf tabs (Internal, USB, Network) and a PSFlycast-style Network tab: draft PR in snes9xPS5, host tests pass, unverified on console. Jason confirmed on the console that playing from a share works. |
| 2026-10-17 | Cheats reload by themselves: a code added to the `.cht` file while a game runs (or with the cheat menu open) is applied without a restart. Draft PR in snes9xPS5, host-tested, console unverified. See [cheats.md](./cheats.md). |
| 2026-10-17 | Second icon after installing a build: the installer, CI and `param.json` use one fixed title ID, so a new build updates `/data/homebrew/PPSA99922` in place; the installer now names every other folder carrying our content ID. Cause on the console not found yet. Draft PR in snes9xPS5, console unverified. See [install-and-icons.md](./install-and-icons.md). |
| 2026-10-17 | Borders around the picture (like SwanStationPS5's bezels): plan, spec and ADR 005 written; split into renderer, pack/lookup and UI/default-art threads. Nothing built. |
