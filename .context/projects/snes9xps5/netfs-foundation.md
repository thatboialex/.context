# netfs foundation (Phase 1, shared layer)

What the first netfs PR on `thatboialex/snes9xPS5` built, and the choices in it. Goal and phases: [network-shares-plan.md](./network-shares-plan.md). Details: [network-shares-spec.md](./network-shares-spec.md). Code: `ps5/netfs/`, tests `ps5/tests/test_netfs.cpp` (group 27).

Status 2026-10-17: draft PR [snes9xPS5#7](https://github.com/thatboialex/snes9xPS5/pull/7). The SMB and NFS backends are separate PRs against `netfs/netfs_backend.h`. Nothing here is wired into the app yet.

## Files

| File | Job |
|---|---|
| `netfs_backend.h` | The contract a protocol backend implements. Frozen: change it only with a note in the PR. |
| `netfs_share.h/.cpp` | `shares.txt` parsing, escaping, mount points, `netfs_redact`, `netfs_error_text`. |
| `netfs.h` | The frontend's API: load, start, mount points, state, last error, realtime flag, test hooks. |
| `netfs_core.h/.cpp` | `Share` (one connection, one lock, request timing), mount jobs, watchdog, routing. |
| `netfs_vfs.h/.cpp` | The libc calls for network paths: handle tables, read buffer, stat, directories. |
| `netfs_wrap.cpp` | `__wrap_*` entry points. The list is `NETFS_WRAPPED` in `ps5/Makefile`. |
| `backend_{smb,nfs}_stub.cpp` | "Not built in" backends. The real backend files replace them in `NETFS_SMB_SRC` / `NETFS_NFS_SRC`. |
| `tests/netfs_fake.h` | In-memory server for tests: counters, `hang`, connect failures. Sibling backends can reuse it. |

## Decisions

1. **Backend paths are relative to the configured location** (SMB: share plus folder; NFS: the whole path after the host). netfs never learns the export/folder split: the NFS backend tries the path, then one folder higher (as SwanStationPS5 does). Why: the mount point maps one to one onto the location, and netfs stays blind to the protocol.
2. **netfs serialises calls per backend; only `Abort()` is concurrent, and it is final.** Why: libsmb2 and libnfs are single-threaded, and the watchdog's kill must not wait for the lock a stuck call holds. A reconnect makes a new backend object.
3. **Handles are slot addresses.** `FILE*` and `DIR*` point into static tables, a descriptor is `0x40000000 + slot`. A handle is netfs's after a range test with no lock. 64 files, 16 folders.
4. **Read-only.** Write modes fail with `EROFS`; folders fail `fopen` with `EISDIR`.
5. **Reads are buffered per file** (64 KiB first, doubling to 512 KiB while sequential; reads of 512 KiB or more bypass the buffer). No whole-file fetch, directory cache or read-ahead yet: those are `netfs_cache` (next).
6. **`netfs_mount_points()` returns only Ready shares**, so a dead share leaves the roots like an unplugged drive. `netfs_shares()` lists all with state and sentence.
7. **Limits:** a connect may run 15 s, any other request 5 s, then the watchdog aborts the connection (`kNetfsConnectLimitMs`, `kNetfsRequestLimitMs`). The clock is injectable.
8. **An address is one word** (spaces are `%20`); only a trailing `# note` may follow. Other trailing text rejects the line, because a space typed into a password would otherwise cut it short without a word.
9. **`user:password@` is rejected** with a hint to use `?password=`: the colon form would silently become part of the user name.
10. **Error sentences scrub the password** (raw and `%XX`) even from library text, as a second line behind the rule that backends never put one in `Status::text`.
11. **Host builds use `-U_FORTIFY_SOURCE`.** Fortify turns `fread`, `fgets`, `read`, `pread` and `realpath` into `__*_chk` calls that `--wrap` does not see.
12. **Left to the backend work:** the `fcntl`/`SO_NBIO` wrap and the `getaddrinfo` shim (spike S2), `config.h` and flags for libsmb2/libnfs (`LIBSMB2_FLAGS`, `LIBNFS_FLAGS` are starting points).

## Unverified

- The PS5 link: `--wrap` through lld with `app-symbols.map`, and that SDK libc.a has every `__real_*` target (`fopen64`, `fseeko64`, `ftello64`, `pread`, `realpath` among them). The cloud has no PS5 SDK; the first CI run of `make ps5` / `make app` is the proof (spike S1).
- Everything on hardware (plan rules 1-7).

## Next

1. `netfs_cache`: whole-file fetch with read-ahead, directory and attribute cache, negative lookups (spec "Reads", "Directory and attribute cache").
2. Wire into the app: `netfs_load("<root>/shares.txt")` and `netfs_start()` at boot, `OrbisRomRoots()` appends `netfs_mount_points()` (Phase 2).
3. SMB backend, then NFS backend (parallel PRs).
