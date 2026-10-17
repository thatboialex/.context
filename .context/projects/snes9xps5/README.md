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

## Files

| File | Load when |
|---|---|
| [architecture.md](./architecture.md) | You touch ROM discovery, loading, saves, the build or the tests. |
| [swanstation-nfs-reference.md](./swanstation-nfs-reference.md) | You implement or review anything copied in spirit from SwanStationPS5. |
| [network-shares-plan.md](./network-shares-plan.md) | You work on SMB/NFS loading: goal, design, phases, open decisions. |
| [network-shares-spec.md](./network-shares-spec.md) | You write the code: address format, per-file changes, tests, risks. |
| [nfs-backend.md](./nfs-backend.md) | You touch `ps5/netfs/backend_nfs.cpp` or the vendored libnfs. |
| [decisions/001-network-rom-loading.md](./decisions/001-network-rom-loading.md) | You question why netfs is built the way it is. |
| [box-art-lookup-plan.md](./box-art-lookup-plan.md) | You work on cover lookup for renamed or fan-translated ROMs: goal, design, acceptance. |
| [decisions/002-box-art-name-matching.md](./decisions/002-box-art-name-matching.md) | You question why art names are matched by words and the ROM header. |

## Status

| Date | State |
|---|---|
| 2026-10-17 | Network shares: plan written, no code. Next step is Phase 0 (spikes) in the plan. |
| 2026-10-17 | Box art lookup for translated ROMs: implemented in a draft PR (words, extra CRC table, ROM header); host tests pass, console unverified. Covers for games libretro lacks now come from an embedded Hasheous/TheGamesDB table (step 7, 411 games), and `Front Mission - Gun Hazard` no longer gets `Front Mission`'s cover; a live per-ROM lookup stays open decision D4 and is not built. |
