# netfs foundation (Phase 1 and 2: shared layer, cache, app wiring)

What the first netfs PR on `thatboialex/snes9xPS5` built, and the choices in it. Goal and phases: [network-shares-plan.md](./network-shares-plan.md). Details: [network-shares-spec.md](./network-shares-spec.md). Code: `ps5/netfs/`, tests `ps5/tests/test_netfs.cpp` (group 27).

Status 2026-10-17: merged into main as [snes9xPS5#7](https://github.com/thatboialex/snes9xPS5/pull/7), followed by the NFS backend (#6), the SMB backend (#8) and the shares screen (#10), all against `netfs/netfs_backend.h`. The cache and the app wiring (boot, shelf, menu, launch) are in #7; the feature works end to end on the host (test groups 28, 29). Nothing has run on the console.

## Files

| File | Job |
|---|---|
| `netfs_backend.h` | The contract a protocol backend implements. Frozen: change it only with a note in the PR. |
| `netfs_share.h/.cpp` | `shares.txt` parsing, escaping, mount points, `netfs_redact`, `netfs_error_text`. |
| `netfs.h` | The frontend's API: load, start, mount points, state, last error, realtime flag, test hooks. |
| `netfs_core.h/.cpp` | `Share` (one connection, one lock, request timing), mount jobs, watchdog, routing. |
| `netfs_cache.h/.cpp` | Per share: folder listings, 512 KiB blocks with read-ahead, the MSU-1 prefetch, the realtime rules. |
| `netfs_vfs.h/.cpp` | The libc calls for network paths: handle tables, stat, directories, reads through the cache. |
| `netfs_wrap.cpp` | `__wrap_*` entry points. The list is `NETFS_WRAPPED` in `ps5/Makefile`. |
| `backend_{smb,nfs}_stub.cpp` | "Not built in" backends. The real backend files replace them in `NETFS_SMB_SRC` / `NETFS_NFS_SRC`. |
| `tests/netfs_fake.h` | In-memory server for tests: counters, `hang`, `gate` (a stalled link), connect failures. Sibling backends can reuse it. |
| `host/netfs_dirbackend.cpp` | Host build only: with `SNES9X_HOST_NETFS_DIR` every share address is a local folder (`..._DELAY_MS` slows reads). Test group 28 runs the real app over it. |

## Decisions

1. **Backend paths are relative to the configured location** (SMB: share plus folder; NFS: the whole path after the host). netfs never learns the export/folder split: the NFS backend tries the path, then one folder higher (as SwanStationPS5 does). Why: the mount point maps one to one onto the location, and netfs stays blind to the protocol.
2. **netfs serialises calls per backend; only `Abort()` is concurrent, and it is final.** Why: libsmb2 and libnfs are single-threaded, and the watchdog's kill must not wait for the lock a stuck call holds. A reconnect makes a new backend object.
3. **Handles are slot addresses.** `FILE*` and `DIR*` point into static tables, a descriptor is `0x40000000 + slot`. A handle is netfs's after a range test with no lock. 64 files, 16 folders.
4. **Read-only.** Write modes fail with `EROFS`; folders fail `fopen` with `EISDIR`.
5. **Reads go through the cache.** A file is 512 KiB blocks (a `.pcm` 128 KiB). Two blocks read in order start a worker thread that fetches the rest of a file up to 16 MiB (nothing evicted) or a window of 8 blocks ahead and 2 behind above that (`.pcm`: 32 ahead). A zip's directory read at the end costs one block and starts no worker.
6. **`netfs_mount_points()` returns only Ready shares**, so a dead share leaves the roots like an unplugged drive. `netfs_shares()` lists all with state and sentence.
7. **Limits:** a connect may run 15 s, any other request 5 s, then the watchdog aborts the connection (`kNetfsConnectLimitMs`, `kNetfsRequestLimitMs`). The clock is injectable.
8. **An address is one word** (spaces are `%20`); only a trailing `# note` may follow. Other trailing text rejects the line, because a space typed into a password would otherwise cut it short without a word.
9. **`user:password@` is rejected** with a hint to use `?password=`: the colon form would silently become part of the user name.
10. **Error sentences scrub the password** (raw and `%XX`) even from library text, as a second line behind the rule that backends never put one in `Status::text`.
11. **Host builds use `-U_FORTIFY_SOURCE`.** Fortify turns `fread`, `fgets`, `read`, `pread` and `realpath` into `__*_chk` calls that `--wrap` does not see.
12. **Listings answer lookups.** `stat`, `lstat`, `access` and `open` of a child come from the parent's listing (60 s); a name missing from a fresh listing is `ENOENT` with no round trip; a name that differs only in case matches when exactly one does (SMB ignores case). In realtime a listing never expires and is never fetched. `netfs_rescan()` forgets them.
13. **Open is lazy.** `fopen` of a file in a listing makes no server call; the server's `Open` happens at the first block fetch, so a refused open shows as a read error.
14. **Realtime (`netfs_set_realtime`) means no waiting on the emulation thread.** Unknown names are `ENOENT`, a `.pcm` read that is not in memory is full-length silence (counted, logged at the 1st and every 100th), a `.pcm` opens only when its head is here (the open queues it for next time), handle closes go to the worker. The one wait: a miss inside an MSU-1 data file that did not fit the 32 MiB prefetch budget (counted as `data_stalls`).
15. **`netfs_prepare_game(rom)`** (before `LoadROM`) lists the ROM's folder, fetches `<rom>.msu` / `msu1.rom` whole, the first 128 KiB of up to 64 tracks (waits up to 3 s for the heads) and pins each track's loop-point block. `netfs_end_game()` releases them. The ROM itself is not prefetched: the loader's sequential reads start the read-ahead.
16. **No 64-bit wraps on the console.** The SDK's libc has no `fopen64`, `fseeko64`, `ftello64`; wrapping them left `__real_*` undefined at link. minizip maps them to the plain calls on FreeBSD targets (`unzip/ioapi.h`). `NETFS_WRAPPED_LFS64` (glibc only) also carries `stat64 lstat64 fstat64 readdir64 open64 pread64 lseek64`: on the host, code that includes `ioapi.h` before the system headers gets those names.
17. **The frontend, not `OrbisRomRoots()`, adds network roots.** `fe::RomRoots()` = `OrbisRomRoots()` + `netfs_mount_points()`, because `orbis_paths.cpp` is also linked into the helper and installer, which have no netfs. The shelf scans at once, then every 0.5 s compares the connected mount points with those it scanned and, on a change, scans again keeping the selected game (`fe_shelf.cpp`). Games on a share (`GameInfo::on_network`) stand on the shelf's Network tab ([ADR 004](./decisions/004-shelf-tabs-network-tab.md); the `NET` badge is gone).
18. **Network ROMs are named without reading them:** file name chain, plus a CRC already in `crc-cache.txt`. `CrcCache::Save()` keeps `/smb/` and `/nfs/` entries. Reading every ROM over the network for a CRC is what would make a scan slow. Games left with only a file-name guess are `crc_pending`; `fe::NetNaming` (a local of `Shelf()`) reads them one at a time on a worker, writes each CRC to the shared cache (saved every 20), and stops when the shelf is left or a game starts (`netfs_realtime`). The shelf scans again when the pass is done, or a minute after the last look, keeping the selection.
18a. **Covers for games that appear late:** the shelf refresh calls `CoverService::Start` again with the new list (host, and the console when the helper downloads in the background), and on the console without a background helper `CoversRestartIfNeeded` writes `wanted.txt` including them, so the next start's prefetch has the covers before the shelf (it restarts once at most, with the existing two-minute guard).
18b. **A lost connection is noticed at once, not only by the watchdog.** After a failed Stat/List/Open/Read, `Share::CheckLost` marks the share Dead (event "connection lost", no longer a root) for `ErrKind::Closed` and for `ENOTCONN ECONNRESET ECONNABORTED EPIPE ENETDOWN ENETUNREACH EHOSTUNREACH ETIMEDOUT`. `EIO` is ambiguous (a dead link or one unreadable file), so it is checked with a stat of the location: only if that fails too is the share lost. (Codex review of PR #7.)
18c. **Case:** the cache's case-insensitive name fallback applies to SMB only; NFS names are exact.
19. **Notices:** `netfs_take_events()` queues "Connected to ...", the failed-mount sentence, "The connection to ... was lost."; the shelf and the game loop (every 60 frames) show them with `ProsperoNotify` and log them as `[net]` in `boot.log`. A failed launch of a network ROM shows `netfs_last_error()` in the message box.
20. **Left to the backend work:** the `fcntl`/`SO_NBIO` wrap and the `getaddrinfo` shim (spike S2), `config.h` and flags for libsmb2/libnfs (`LIBSMB2_FLAGS`, `LIBNFS_FLAGS` are starting points).

## Unverified

- The PS5 link: `--wrap` through lld with `app-symbols.map`, and that SDK libc.a has every `__real_*` target (`fopen64`, `fseeko64`, `ftello64`, `pread`, `realpath` among them). The cloud has no PS5 SDK; the first CI run of `make ps5` / `make app` is the proof (spike S1).
- Everything on hardware (plan rules 1-7).

- Block sizes, the 16 MiB whole-file limit, the 32 MiB budget and the 3 s head wait are starting values from the spec; Phase 4 measures them.
- Covers on the console for games that appear after the first scan rely on `CoversRestartIfNeeded` (restart, then prefetch) or the background helper; only the host path (the shelf's own downloads) is tested.

- MSU-1 packs (`<rom>.msu1`, a zip read by minizip) on a share are not prefetched and are cached as ordinary data, so a miss waits for the network and can pause the game. Loose `.msu`/`.pcm` files are the supported layout (README says so). Fixing it means reading the zip's directory and prefetching each entry's head, or refusing the pack in realtime; neither is in this PR.

## Next

1. SMB backend and NFS backend (parallel PRs), then S4 (a real read on the console) and S5 (how much can be allocated).
2. A Network page in Settings (state, last error, "Rescan"): postponed, there is no on-screen keyboard yet and `shares.txt` is the v1 way.
3. README section for players (shares.txt, SMB, NFS): written, in the root README.
