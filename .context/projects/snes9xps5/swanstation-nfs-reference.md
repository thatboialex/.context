# Reference study: SwanStationPS5 "Games on an NFS server"

Source: `darkxex/SwanStationPS5`, commit `de571cf` (1.4.0), read 2026-10-17. License GPL-3.0-or-later. A PS1 emulator for the same family of jailbroken-PS5 native apps. Paths below are inside that repo.

## What it does

| Fact | Detail |
|---|---|
| Protocol | **NFS only** (libnfs 18.0.0, LGPL-2.1+, vendored in `third_party/libnfs`). There is no SMB anywhere in the repo. |
| User view | Settings > System > NFS shares. Type `nfs://<ip>/<export>` (several separated by `;`). Games appear on the shelf like a USB drive's. Read-only. Saves, states, covers stay on the console. |
| Options | After `?`: `version=4`, `nfsport=`, `mountport=`, `uid=`, `gid=`. NFS v3 by default, with the server's portmapper. The path may be inside an export. |
| Startup | Opening the app never waits for a server. Shares mount on a background thread (8 s limit each) and the library is scanned again when they are ready (`src/main.c:2705`). |
| Recovery | Settings > Library > Rescan library re-checks shares and mounts missing ones. No automatic retry. |
| Connection lost in a game | The game is left, the library reloads, a notice says "NFS communication was lost" (`main.c:2699`). |
| Errors | Mount failures are classified into nine reasons (`NFS_FAIL_*`, `src/nfs/nfs.h`) and shown in plain words. When an export is wrong, the notice lists what the server does export. |

## How it works

| Piece | Where | Idea |
|---|---|---|
| Virtual paths | `src/nfs/nfs.cpp:273-331` | A share appears under `/nfs/<host>/<path>`. The emulator core and the frontend open those paths like normal files. |
| Interception | `Makefile:31-34` `APP_WRAP_SYMBOLS` | Linker `--wrap` redirects `fopen fclose fread fseek fseeko ftell ftello rewind feof ferror fileno fgetc getc fgets fflush setvbuf flockfile funlockfile open close read pread lseek fstat stat lstat access realpath opendir readdir closedir` to `__wrap_*` in `nfs.cpp` (line 1235 on). A path that is not virtual goes to `__real_*`. |
| Fake handles | `nfs.cpp:137-184` | The `FILE*` it returns is a pointer into a static `File[128]` array; descriptors start at `0x6e660000`. Any FILE function it forgot to wrap would crash on such a pointer. |
| Single lock per share | `nfs.cpp:97-106` | libnfs contexts are not thread safe, so one mutex per share serialises all calls. |
| Block cache | `nfs.cpp:89-95, 119-135` | 512 KiB blocks, `std::map` per file. Total cap 128 MiB (`kCacheTotal`), at least 32 MiB per file. The README says 256 MiB; the code and the 1.4.0 commit say 128 MiB. |
| Read-ahead | `nfs.cpp:604-652` | Only files over 16 MiB get a worker thread. It reads the block the game wants first, then the next ones up to 7/8 of the file's cache share. `trim()` drops blocks farthest from the reader, counting blocks behind it double. |
| Blocking reader | `nfs.cpp:655-703` | A miss makes the game's thread wait (100 ms condition-variable loops) until the block is there. Correct for a CD drive; not acceptable for a 60 fps loop. |
| Watchdog | `nfs.cpp:435-470` | Every 250 ms a detached thread checks each share's in-flight read. Over 5 s: mark the share dead, set a "lost" flag, `shutdown()` the socket so the blocked libnfs call fails. |
| Mount job | `nfs.cpp:862-955` | Up to 8 attempts, each retried one folder higher when the path is not exported. On failure it asks the server for its export list for the error text. |
| Mount all | `nfs.cpp:1049-1148` | Probes already-mounted shares first (a restarted server forgets old file handles), but only while no file is open. Then mounts the rest in parallel with a deadline. |
| Sidecar files | `nfs.cpp:232-252` | The core wants to write `.sbi` files beside the disc; those go to a local overlay folder. Snes9x writes nothing beside ROMs, so this is not needed here. |
| Persistence | `src/config.c:285`, `settings_screen.c:315,1832-1912` | One string setting `nfs_shares`, edited through the app's own on-screen keyboard (`ui/textinput_screen.c`, 213 lines). There is also a phone web page for settings (`src/remote.c`). |

## Console quirks it works around

These apply to any socket-using native title. Verify each in Phase 0 of the plan.

| Quirk | Fix in SwanStationPS5 |
|---|---|
| `getaddrinfo` imports from a module a native title does not load, so a call jumps to 0. | Own `getaddrinfo` on `sceNetResolver*`, IPv4 only (`examples/update-check/console_curl.c:160-230`, boilerplate by BlackBearReloaded, GPL-3.0+). |
| `fcntl` on sockets is refused in a sandboxed title. | `--wrap=fcntl`; non-blocking sockets use the console's `SO_NBIO` (`0x1200`). `fcntl` is first in `APP_WRAP_SYMBOLS`. |
| `SIGPIPE` would end the app. | `third_party/libnfs/config_ps5.h` leaves out `signal()`; `lib/socket.c:154` sets `SO_NOSIGPIPE` on each socket under `__PROSPERO__`. |
| No `getprotobyname`, no `getservbyport`. | libnfs patched under `__PROSPERO__` to use `IPPROTO_TCP` and skip the services lookup (`socket.c:215`, `1993`). |
| `clock_gettime` lacks `CLOCK_MONOTONIC_COARSE`; no `pwd.h`. | `config_ps5.h` maps it to `CLOCK_MONOTONIC` and turns `HAVE_PWD_H` off. |
| The C heap a native title gets is small ("OOM for vout_buf"). | Own `malloc` family on a big direct-memory block (`src/platform/ps5_heap.c`, dlmalloc). Needed before a 128 MiB cache works. |
| Hung synchronous calls. | The watchdog above. libnfs has no limit that holds when a server vanishes mid-read. |

The build (`tools/build-libnfs.sh`) compiles `lib mount nfs nfs4 portmap` sources into `libnfs.a` with `-DHAVE_CONFIG_H -DHAVE_NFS4_2`.

## What carries over to snes9xPS5

| Keep (the design) | Change (because SNES differs) |
|---|---|
| Linker `--wrap` so the core stays unchanged. | Wrap `fopen64 fseeko64 ftello64` too, or build minizip with `-DMINIZIP_FOPEN_NO_64` (`unzip/ioapi.c:18`). SwanStation's list lacks them. |
| Virtual `/nfs/...` paths, one lock per share, fake handles. | Add `/smb/...` and a backend interface. |
| Background mount, rescan when ready, never block startup. | Also keep the shelf selection when the network games arrive. |
| Watchdog that kills a hung socket. | Losing the connection must not end a game whose ROM is already in RAM. |
| Plain-word error classification, export listing. | Add SMB reasons; list shares with `smb2_share_enum_sync` when a share name is wrong. |
| Block cache with read-ahead. | Use it for big files only (MSU-1). ROMs are at most 16 MiB: fetch the whole file at open. |
| Blocking reader. | Emulation-thread reads must not wait on the network more than a few ms (see plan, "Smooth playback contract"). |

## Licence note

`nfs.cpp` and `console_curl.c` are GPL-3.0-or-later. `ps5/` code in snes9xPS5 is MIT. Take the design, not the code ([ADR 001](./decisions/001-network-rom-loading.md)). libnfs and libsmb2 are LGPL-2.1 and can be vendored with their notices. Credit darkxex/SwanStationPS5 for the design in the README when the feature ships.
