# ADR-001: Load ROMs from network shares through a linker-wrapped netfs layer

## Status
Proposed, 2026-10-17. Becomes Accepted when Jason approves the plan PR and Phase 0 passes.

## Context

Jason wants Snes9x PS5 to play ROMs from a drive behind a network server, with smooth gameplay, modelled on SwanStationPS5's "Games on an NFS server". Study of that app ([swanstation-nfs-reference.md](../swanstation-nfs-reference.md)) and of this repo ([architecture.md](../architecture.md)) shows:

- SwanStationPS5 supports NFS only. SMB needs a second library (libsmb2).
- The console has no network filesystem client. The app must carry its own.
- The Snes9x core opens ROMs, patches and MSU-1 files with plain `fopen`, `gzopen` and minizip, and the repo rule is to leave the core unchanged.
- A ROM is at most 16 MiB and is copied into RAM at launch. Only MSU-1 files are large and read during play.
- `nfs.cpp` in SwanStationPS5 is GPL-3.0-or-later; new code in `ps5/` is MIT.

## Decision

1. Add a `netfs` layer in `ps5/netfs/` that serves virtual folders `/smb/<host>/<share>/...` and `/nfs/<host>/...`. It intercepts libc file calls with the linker's `--wrap`, so the core, minizip and zlib are untouched.
2. Put the protocols behind one backend interface: libsmb2 for SMB, libnfs for NFS. SMB ships first (Jason's stated need); NFS follows. If the libsmb2 spike fails on the console, NFS goes first.
3. Make ROM loading smooth by fetching the file whole through an adaptive read-ahead (window = whole file up to 16 MiB). After load the network is idle. Use the streaming cache only for large files (MSU-1), with a realtime mode in which `.pcm` audio reads return silence instead of blocking the emulation thread. MSU-1 `.msu` data is never skipped.
4. Share the connection watchdog design (kill the socket when a request runs over 5 s), but a lost connection never ends a running game.
5. Reimplement from the design. Do not copy SwanStationPS5 source. Vendor libsmb2 and libnfs (LGPL-2.1) with notices, and credit SwanStationPS5 in the README.
6. Shares are configured in `/data/snes9x/shares.txt`, read-only on the share side. An on-screen keyboard comes later.

## Alternatives considered

| Option | Why not |
|---|---|
| Edit the core's `OPEN_FSTREAM` macros and `FileLoader` | Breaks the "core unchanged" rule, makes upstream merges harder, and still misses minizip and the `gzopen` paths. |
| Copy the ROM to `/data` first, then load it | Simple, but a long wait before every launch, uses console storage, and cannot serve 100s of MB of MSU-1 audio. It stays a possible optional cache (plan D5). |
| Adapt SwanStationPS5's `nfs.cpp` directly | Fastest for NFS, but GPL-3.0 code inside MIT files, and it blocks the reader on misses. Revisit only with darkxex's permission. |
| Mount the share in the kernel | The console has no NFS client (SwanStationPS5 README) and nothing shows one for SMB. An app cannot add a kernel mount. |
| SMB through the helper payload | Adds an IPC copy of every byte and a second place for credentials, with no smoothness gain. |

## Consequences

### Positive
- Core stays identical to upstream; the feature is contained in `ps5/`.
- Smooth play is a property of the design (no network after load), not of tuning.
- SMB reaches Windows PCs, NAS boxes and most router USB shares; NFS reaches Linux and NAS.
- A fake backend makes the hard cases (hangs, latency, loss) testable in CI without a console.

### Negative
- `--wrap` is fragile: a libc file call missing from the list crashes on a fake `FILE*`. The list needs a review whenever new file code lands.
- Two vendored LGPL libraries, each with PS5 patches to maintain.
- Cannot be fully verified in the cloud; the console checklist depends on Jason.
- Reimplementing costs more than adapting `nfs.cpp`.

### Neutral
- A scan of a network share costs listing time and a deferred CRC pass, so titles for oddly named ROMs appear a little after the shelf does.
- The README gains a setup section for servers (Windows, Samba, NAS, router).
