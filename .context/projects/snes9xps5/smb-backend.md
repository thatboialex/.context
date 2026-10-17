# SMB backend (libsmb2): what it does, what it assumes

Status: code written, host-tested against Samba, **not run on a console**. Code: `ps5/netfs/backend_smb.cpp`, `ps5/third_party/libsmb2/`. Contract it implements: `ps5/netfs/netfs_backend.h`. Plan: [network-shares-plan.md](./network-shares-plan.md). Load this file when you touch the SMB backend or debug an SMB share.

## Split of work with netfs

| netfs (core) | SMB backend |
|---|---|
| shares.txt parsing, redaction, mount points, libc `--wrap` | takes a parsed `ShareConfig`, never sees the address line |
| block cache, whole-file fetch, MSU-1 read-ahead, silence on late `.pcm`, lossless `.msu` | `Pread` that is fast, bounded and cancellable; nothing is cached here |
| watchdog (5 s) and the "connection lost" event | `Abort()` and its own silence limit (4 s) |

## Decisions

1. **libsmb2 6.2, vendored from the Debian `orig` tarball** (client sources only, no Kerberos, no CMake). `config.h` is ours. On the console `prospero-clang` defines `__FreeBSD__`, so libsmb2 takes its FreeBSD branch (`<sys/endian.h>`), not `PS4_PLATFORM`. CI compiles and links it that way (PR snes9xPS5#8); how it behaves on a console is unverified.
2. **Name lookup happens in the backend**, once: a dotted address is used as is, otherwise `getaddrinfo` (IPv4). libsmb2 is then given the dotted address, so it never resolves anything itself. On the console `getaddrinfo` is the core's `--wrap` shim (libSceNet), because the native one is broken (plan, Phase 0).
3. **Non-blocking sockets** are libsmb2's own `fcntl(O_NONBLOCK)` call, which the core's `--wrap` shim turns into `SO_NBIO` on the console (`ps5/netfs/netfs_shims_ps5.cpp`). The backend has no console code of its own. The `SO_NBIO` value is unverified there.
4. **Own wait loop, async API.** Every call polls in 100 ms slices, so `Abort()` (a flag) is seen within 100 ms and no socket is shut down from another thread (no fd-reuse race). A call fails after 4 s of silence. A dead connection is never reused; `Disconnect()` destroys it.
5. **Reads are pipelined**: a `Pread` splits into 64-256 KiB requests, up to 8 in flight. A 64 KiB-per-request server (SMB 2.02, many routers) then costs a few round trips per 512 KiB block, not eight. The file size comes from the open reply, so a 0-byte answer inside the file is retried, not taken for the end.
6. **Identity.** No user in the address means guest: "Guest" with an empty password (Windows, Samba), then an anonymous session (some NAS boxes). A user always uses exactly the user and password given. Library text that becomes a player-visible message is scrubbed of the password.
7. **Errors the player sees** come from NT status codes and errno, not from library wording: wrong password -> `AccessDenied`, no such share -> `ShareNotFound` (+ the shares the server offers, over a second connection to `IPC$`), no folder -> `FolderNotFound`, refused / silent / closed / unknown name map directly. **SMB1-only** is proven, not guessed: after an unexplained failure the backend sends one SMB1 negotiate and checks for an SMB1 reply.

## Limits (v1)

IPv4 only. IP addresses and DNS names only (no NetBIOS or mDNS names). SMB 2/3 only. No reconnect and no keep-alive: an idle share that the server dropped fails the next call with `EIO` and netfs marks it lost. Symlinks are whatever the server resolves.

## Tests

`make -C ps5 test` runs `tests/smb/run.sh` after the usual groups. It starts a userspace Samba on the loopback (skipped when `smbd` is not installed) in four setups: SMB 2/3 with guest and user, signing mandatory, encryption required, SMB1 only. A TCP proxy in front can add latency, hang or cut the link. Covered: connect, list (300 files, UTF-8 name), stat, byte-identical 6 MiB read, edges (tail, past the end, empty file), SMB 2.02, 50 ms link, wrong password (and no password in the message), wrong share, wrong folder, refused port, unknown name, silent address, hung link, `Abort()` from another thread, cut link, and a leak check (ASan).

## Console checklist (Jason, SMB only; the generic list is in the spec)

1. The app starts with the SMB backend linked in (CI already builds the ELF; this is the first run).
2. `shares.txt` with an IP address: connect, list, launch a ROM.
3. The same with a DNS name (`nas`, `nas.lan`): the resolver path.
4. Guest, then a user with a password containing `@ & = %`.
5. Pull the cable mid-game: the game keeps running; a MSU-1 track goes silent.
6. A router's USB share: note the SMB version in the notice (or the SMB1 message).
