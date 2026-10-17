# Spec: network shares for snes9xPS5

For the person writing the code. Goal, phases and decisions are in [network-shares-plan.md](./network-shares-plan.md). Numbers marked "start" are first guesses to tune after the spikes.

## Share addresses

File: `/data/snes9x/shares.txt`. One address per line, `#` starts a comment, at most 8 shares. The app creates a commented template when the file is missing and never rewrites a file it did not create (until Phase 5 adds editing, which writes atomically through `.part`).

```
smb://192.168.1.50/games/snes                            guest
smb://jason@192.168.1.50/games/snes?password=secret      user and password
smb://WORKGROUP;jason@nas/games?password=x&vers=3&sign   domain, SMB 3, signing
nfs://192.168.1.50/export/snes                           NFS v3
nfs://nas/export/snes?version=4&uid=1000&gid=100
```

| Scheme | Options (after `?`, joined by `&`) |
|---|---|
| `smb://[domain;][user@]host[:port]/share[/folder]` | `password` (ours), `vers` (`2`, `3`, `2.02`, `2.10`, `3.00`, `3.02`, `3.1.1`), `sign`, `seal`. `vers`, `sign` and `seal` use libsmb2's URL names. |
| `nfs://host[:port]/export[/folder]` | `version` (`3` or `4`), `nfsport`, `mountport`, `uid`, `gid`. Same meaning as SwanStationPS5. |

Parse these ourselves and call `smb2_set_*` / `nfs_*` directly. Never hand the password to a URL parser that might log it.

**Escaping.** Addresses use URL percent-encoding. A password, user, domain, folder or option value that contains `& = % # @ ; / ?`, a space or a non-ASCII character is written as `%XX` bytes (UTF-8). For example the password `p@ss&word=1` is written `p%40ss%26word%3D1`.

- Split the query on `&`, then each option on its first `=`, and only then percent-decode. An encoded delimiter therefore never splits an option.
- A line is a comment only when its first non-space character is `#`. A `#` inside an address belongs to it.
- A bad escape (`%zz`) rejects the line. The message names the line number, never its text.

**Redaction.** `netfs_redact()` builds its text from the parsed fields: scheme, domain, user, host, port, share and folder, plus the non-secret options `vers`, `sign` and `seal`. It never edits the raw line, so no encoding of the password can slip through. Logs and notices show only that output, or the line number for a line that did not parse.

Virtual paths: `/smb/<host>/<share>/<folder>` and `/nfs/<host>/<folder>`. Two shares with the same mount point are rejected with a notice.

## Reads

One block cache for every file (start: 512 KiB blocks, below the SMB and NFS maximum read size).

1. **Open.** Resolve the share, get size and mtime from the cached directory listing (one `stat` round trip on a miss).
2. **Adaptive read-ahead.** A worker thread starts after two sequential block reads. Its window is the whole file when the file is at most 16 MiB, and nothing is evicted. A ROM loaded by `FileLoader` is therefore fetched at wire speed. A zip whose directory is read (seek to the end) fetches one or two blocks and no worker starts.
3. **Big files** (MSU-1 audio): window is a share of the cache cap (start: 16 MiB total, set after spike S5). The blocks farthest from the reader are dropped first, blocks behind it count double. Block 0 of every open file is pinned so loops to the start cost nothing.
4. **Realtime mode.** The frontend calls `netfs_set_realtime(true)` after `LoadGame` returns and `false` in `CloseGame`. Launch and scan may wait for data (bounded by the watchdog). In realtime mode nothing on the emulation thread waits on the network, with one exception (item 6):
   - **`.pcm` audio** (`<rom>-<n>.pcm`): a read that misses returns the full requested span as zero bytes and advances the position, and `underruns` goes up. Never return a short read: the core treats one as the end of the track and loops or stops it (`msu1.cpp:241-270`).
   - **Opens** are answered from memory only. If the track's head is not cached, the open fails; the core then sets `AudioError` (`msu1.cpp:111`), the log says which track, and the fetch is queued.
   - **Everything else** (ROM, zip, patch) is already loaded before realtime starts.
5. **MSU-1 prefetch.** Before the first frame (launch may wait, deadline start: 3 s), list the ROM's folder from the cached listing and fetch the first 128 KiB of every `<rom>-*.pcm` (start: at most 64 tracks, 8 MiB). `msu1.cpp:119-125` checks the `MSU1` header at open. When a track opens, the worker also fetches and pins the block holding its loop point, `8 + 4 * le32(bytes 4..7)` (`msu1.cpp:125-129`), so loops are seamless.
6. **MSU-1 data** (`<rom>.msu`, `msu1.rom`) is game data read through port 1 (`msu1.cpp:294-345`). It is never zero-filled or skipped: that would corrupt the game. Fetch it whole before the first frame when it fits the budget (start: 32 MiB for all data files). An oversize data file is served by the block cache with a window around the read position, and a miss there waits for the block (lossless, bounded by the watchdog, counted in `data_stalls`). This is the single documented exception to the no-wait rule. Phase 4 measures real MSU-1 packs to see whether it ever triggers.
7. **Dead share.** The watchdog (every 250 ms) marks a share dead when one request has run over 5 s and shuts its socket. After that reads return `EIO` (`.pcm` reads return silence) and the frontend gets one "connection lost" event.
8. **Locks.** One mutex per connection around every library call. Both libraries are single-threaded. Use a second connection for the worker if the spike shows one connection is too slow.
9. **Clock.** All timeouts take an injectable clock so tests never sleep (`.claude/rules/testing.md`, rule 6).

## Directory and attribute cache

- `list(path)` returns names and attributes together (`smb2_readdir` fills `struct smb2dirent::st`; libnfs `nfs_readdir` returns attributes). Cache per folder until rescan or 60 s (start).
- `stat`, `lstat`, `access`, `open` of a child answer from the parent's listing with no round trip.
- A name absent from a fresh listing answers `ENOENT` at once. This covers the patch and MSU-1 probes (`memmap.cpp:3931-4060`, `msu1.cpp:58-100`).
- Never probe network roots with `stat` in `OrbisRomRoots()`; use the in-memory mount-point list.

## Wrapped calls

Start from SwanStationPS5's list and add what Snes9x needs:

`fopen fclose fread fseek fseeko ftell ftello rewind feof ferror fileno fgetc getc fgets fflush setvbuf flockfile funlockfile open close read pread lseek fstat stat lstat access realpath opendir readdir closedir`, plus `fopen64 fseeko64 ftello64` (minizip, `unzip/ioapi.c:23`), plus `fcntl` for socket flags (PS5 build only). Alternative for minizip: build it with `-DMINIZIP_FOPEN_NO_64`. Add the list to the EBOOT link (`ps5/Makefile:224-231`) and to the host test link (line 321). Any FILE function the code uses and the list misses will crash on a fake `FILE*`, so grep `ps5/` and the core for others before freezing the list.

## Library and UI changes (Phase 2)

| File | Change |
|---|---|
| `orbis_paths.cpp:100` | `OrbisRomRoots()` appends `netfs_mount_points()`. |
| `fe_games.h/.cpp` | Add `bool on_network` beside `on_usb`. `Walk()` gets the flag. Network ROMs skip `RomCrc32` in pass 1 (name chain only). A worker resolves their CRCs later and updates titles. `CrcCache::Save()` (line 179) keeps `/smb/` and `/nfs/` entries like `/mnt/`. |
| `fe_shelf.cpp:630, 884` | Scan local roots first, as today. Poll `netfs_mount_finished()` and append network games without resetting the selection. Badge "NET". |
| `fe_menu.cpp:203, 484-545` | `RootLabel` for network roots ("Network: host/share"). The 3 s re-poll uses the mount-point list. |
| `fe_emu.cpp:599-640` | After a failed `LoadROM`, show `netfs_last_error()` in a notice. Call `netfs_set_realtime`. On `CloseGame` stop the head prefetch. |
| New `fe_netscreen.cpp` | Settings page: each share, state, last error, "Rescan". |
| `ProsperoNotify.cpp` | Reuse for "share connected", "share not reachable", "connection lost". |
| `ps5/Makefile` | New `netfs` objects, libsmb2 and libnfs rules (pattern: `PS5_MBEDTLS_OBJ`, line 101), `--wrap` flags, vendored sources in `ps5/third_party/{libsmb2,libnfs}` with their notices. |
| `README.md` | New section, setup for Windows/Samba/NAS/router, credits for libsmb2, libnfs and SwanStationPS5. |

## Errors shown to the player

One sentence each, in the style of SwanStationPS5: no answer; connection refused; server closed the connection; name not found; access denied or wrong password; share not found (list the shares with `smb2_share_enum_sync`); NFS export not found (list exports); folder not on the server; server only speaks SMB1; signing required; other (library text, redacted). Map from `smb2_get_error` text and NT status codes (`LOGON_FAILURE`, `BAD_NETWORK_NAME`, `ACCESS_DENIED`).

## Spikes (Phase 0)

| # | Question | How | Pass |
|---|---|---|---|
| S1 | Does `--wrap` link into `eboot.bin`? | Wrap `fopen` only in the EBOOT link with `app-symbols.map` and the libc.prx manifests unchanged. | Boots; wrapper called. |
| S2 | Do resolver and non-blocking sockets work in the sandboxed app? | `getaddrinfo` replacement on `sceNetResolver` (pattern: `fe_tlshttp.cpp:351-380`), `fcntl` wrap using `SO_NBIO`. | IP literal and DNS name resolve; socket goes non-blocking. |
| S3 | Do libsmb2 and libnfs compile? | `prospero-clang`, `-DPS4_PLATFORM` for libsmb2, libnfs config from SwanStationPS5's `config_ps5.h`. Patch out `getservbyport`, `getprotobyname`, `signal`. | Static archives link. |
| S4 | Does a read work end to end? | From a spike build on Jason's console: connect (guest, then user), list a folder, read a 6 MiB file, print MiB/s. | Bytes match the file's SHA-256. |
| S5 | How much can the app allocate? | `malloc` in 1 MiB steps until failure; try `mmap` and direct memory. | Number recorded; decides cache sizes. |
| S6 | Does `--wrap` work under ASan on the host? | Wrap `fopen` in the Linux test link. | `make -C ps5 test` still passes. |

## Tests (new groups in `ps5/tests/run-tests.sh`, after group 26)

| Group | Backend | Checks |
|---|---|---|
| 27 netfs core | Fake (deterministic latency, drops, hangs; injectable clock) | `.sfc`, headered `.smc`, `.zip`, `.gz` load byte-identical. Zero reads over 600 frames. Hang is cut by the watchdog and the game keeps running. Negative lookups cost no round trip. Passwords containing `& = % # @ ;` parse correctly and never appear in any log, raw or encoded. |
| 28 library | Fake, 2,000 ROMs | Shelf opens with a share that never answers. Network games join without moving the selection. Scan round trips about one per folder. CRC cache keeps `/smb/` entries while the server is off. |
| 29 MSU-1 | Fake, 64 MiB `.pcm` and a `.msu` file | Prefetch serves the open. Latency 200 ms gives full-length silence on `.pcm` reads, never a stall or a short read. `.msu` bytes stay identical under latency. Loss mid-track mutes and notifies once. |
| 30 real servers (optional CI job) | libsmb2's `examples/smb2-server-sync` (userspace) for SMB; a userspace NFS server for NFS | Mount, list, read a ROM, wrong password message, wrong share lists shares. Check availability of the NFS server in Phase 3. |

Unit tests mock the network; integration tests (group 30) use real servers, as the testing rules require.

## Hardware checklist (Jason, per phase)

1. Ethernet and Wi-Fi: a 6 MiB ROM launches; note the time.
2. A game plays 10 minutes with no audio or video hitch while the share is busy (copy a large file to the NAS meanwhile).
3. Unplug the server's cable mid-game: the game keeps running.
4. Wrong password, wrong share, server off: each gives its own message.
5. Reboot the server while the shelf is open, then Rescan.
6. 1,000+ ROM share: shelf opens at once; names fill in.
7. An MSU-1 hack from the share: tracks start, loop, and survive a cable pull with a notice.
8. A Windows share, a Samba or NAS share, and a router USB share (note SMB version).

## Security

Passwords sit in plain text in `shares.txt` on `/data`, readable over FTP. Say so in the README. NTLM and SMB signing protect the wire; NFS trusts the source address, so export read-only to the console's IP. Never log a password; never put one in a notice.
