# snes9xPS5: architecture map for ROM handling

Verified against `thatboialex/snes9xPS5` commit `d917161` (2026-10-17). Line numbers drift; re-check with `rg` before relying on them. Paths are relative to the repo root.

## Overview

Three layers, from the bottom:

| Layer | Where | Rule |
|---|---|---|
| Snes9x core | everything outside `ps5/` | Unchanged from upstream. Do not edit. |
| PS5 layer | `ps5/coreorbis/` | Video, audio, pads, sandbox request, installer, logs. Prefix `Prospero*`. |
| Frontend | `ps5/frontend/` | Shelf, menus, covers, library scan, `fe_emu.cpp` glue to the core. Prefix `fe_`. |

The app runs sandboxed until the helper (`Snes9xPS5-helper.elf`, port 9089) lets it out, then it sees `/data` and `/mnt/usbN`. HTTPS is done by the helper, not the app (the console's own HTTPS fails outside the sandbox).

## How a ROM gets from disk to the screen today

| Step | Code | What it does |
|---|---|---|
| 1. Roots | `ps5/coreorbis/orbis-shims/orbis_paths.cpp:100` `OrbisRomRoots()` | `/data/snes9x/roms`, `/mnt/usb0-7/snes9x/roms`, `/mnt/ext0-1/snes9x/roms`. Existing folders only, probed with `stat`. |
| 2. Scan | `ps5/frontend/fe_games.cpp:406` `ScanGames()` | `Walk()` (line 237) lists up to 4 folder levels with `opendir`/`readdir` plus one `stat` per entry. Extensions: `.sfc .smc .swc .fig .bs .st .zip .gz .bin .mgd`. |
| 3. Name | `fe_games.cpp:419-441` | Chain: exact No-Intro file name, then CRC32, then loose name, then fuzzy name. A CRC found in `data/snes-extra-crc.tsv` (fan translations) sets `crc_name`, and a ROM no table names gets its header title (`fe_romheader.h`); both feed only the art lookup (`fe_covers.cpp` `MatchArtName`, `fe_artmatch.h`). See [box-art-lookup-plan.md](./box-art-lookup-plan.md). |
| 4. CRC | `fe_games.cpp:340` `RomCrc32()` | Reads the whole ROM (zip: reads the stored CRC from the directory). Cached in `/data/snes9x/covers/crc-cache.txt` by path, size and mtime (`CachedCrc`, line 206). |
| 5. Shelf | `ps5/frontend/fe_shelf.cpp:630` | Calls `ScanGames` on the UI thread with a progress callback. USB badge from `GameInfo::on_usb` (`fe_shelf.cpp:884`). |
| 6. Browser | `ps5/frontend/fe_menu.cpp:484-545` | Folder list screen. Calls `OrbisRomRoots()` again every 3 s while on the root list. |
| 7. Load | `ps5/frontend/fe_emu.cpp:599` `LoadGame()` | `Memory.LoadROM(path)` (line 606), then `LoadSRAM`, cheats. |
| 8. Read | `memmap.cpp:1217` `FileLoader()` | Reads the entire file into `Memory.ROM` (`MAX_ROM_SIZE` is 16 MiB, `memmap.h:22`). `.zip` goes through minizip `unzOpen` (`loadzip.cpp:25`); plain and `.gz` through `gzopen` because the build defines `-DZLIB` (`snes9x.h:20`, `ps5/Makefile:67`). |

After step 8 the ROM is in RAM. Nothing reads the ROM file again during play.

## File reads that stay open or repeat during play

| Feature | Code | Pattern |
|---|---|---|
| MSU-1 audio | `msu1.cpp:232-266` | Opens `<rom>-<track>.pcm` when the game picks a track. Then reads 4 bytes per sample on the emulation thread (`READ_STREAM`), seeks to loop points (`REVERT_STREAM`). Files are 10s to 100s of MB. |
| MSU-1 data | `msu1.cpp:294-345` | Opens `<rom>.msu` or `msu1.rom`. Byte reads (`GETC_STREAM`) after seeks. |
| MSU-1 pack | `msu1.cpp:72-90` | `<rom>.msu1` zip pack looked up in the ROM folder, then the patch folder. |
| Patches | `memmap.cpp:3931-4060` | At load, probes the ROM folder for `.bps .ups .ips`, `NNN.ips`, `ips%d`, `ip%d`. Each miss is a failed open. |
| Multi-cart, BS-X | `memmap.cpp:1612-1700` | Extra ROMs and BIOS read at load. BIOS comes from `/data/snes9x/bios`. |

## Where files are written

`S9xGetDirectory()` (`fe_emu.cpp:433`) sends SRAM, states, cheats, patches, screenshots and logs to `/data/snes9x/<kind>`. Only `ROMFILENAME_DIR` and `ROM_DIR` follow the ROM's folder, and they are read-only uses. A read-only share therefore works without changes to save handling.

## Build and test

| Topic | Fact |
|---|---|
| PS5 build | `make -C ps5 ps5`. Compiles with the payload SDK's `prospero-clang++`; the app links with `$(LLD)` against the SDK's static `libc.a`, `libc++` (`ps5/Makefile:224-231`). `PS5_PAYLOAD_LIBS` (line 97) is for the helper. |
| libc surface | `ps5/proto/native/tooling/native/runtime/api-surface.txt` and `imports.txt` are byte-identical to SwanStationPS5's. Anything SwanStationPS5 links (sockets included) resolves here too. |
| Host tests | `make -C ps5 test` builds Linux binaries with ASan/UBSan. `ps5/host/sce_host.cpp` fakes the PS5 calls. `ps5/tests/run-tests.sh` has groups 1-26 (last: line 841). |
| CI | `.github/workflows/build.yml`: `host-tests` and `ps5-installer` (uploads the ELF as an artifact). |
| Third-party code | Vendored: `ps5/third_party/zlib`, `ps5/frontend/third_party/mbedtls`, stb headers. Credits live in the root `README.md` ("License and credits"). |
| Settings | `snes9x-ps5.ini` (`fe_settings.cpp`). Known keys only; rewritten atomically on save. |
| Text input | None. There is no on-screen keyboard. |

## Sockets today

- The sandboxed app only opens blocking TCP connections to `127.0.0.1` (the helper): `ProsperoJailbreak.cpp:33-50`. It makes no LAN connections.
- The helper payload (a different process context) does LAN-facing sockets in `fe_tlshttp.cpp`: `getaddrinfo`, then a `sceNetResolver` fallback (lines 351-380), and `fcntl(O_NONBLOCK)` (lines 473-495). That is working MIT code to learn from, but it has not been proven inside the sandboxed app.
- SwanStationPS5 shows that LAN TCP from an app of this kind works, with workarounds for `getaddrinfo` and `fcntl` ([swanstation-nfs-reference.md](./swanstation-nfs-reference.md), "Console quirks").
