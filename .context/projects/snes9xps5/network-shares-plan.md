# Plan: play ROMs from network shares (SMB and NFS)

Status: Proposed, 2026-10-17. Requested by Jason. Code: none yet. Details for the implementer: [network-shares-spec.md](./network-shares-spec.md). Why the design is shaped this way: [ADR 001](./decisions/001-network-rom-loading.md).

## Goal

Add a share in Snes9x PS5 and play ROMs from a drive that sits behind a network server (a PC, a NAS, a router's USB port) as smoothly as from `/data`. SMB first, because PCs, NAS boxes and routers speak it. NFS second, because it is the feature SwanStationPS5 ships.

A USB drive plugged into the console already works (`/mnt/usbN/snes9x/roms`). This plan is for drives you reach over the network.

Non-goals: writing to shares, SMB1, Kerberos, server discovery, netplay.

## What the study found

1. **SwanStationPS5 does NFS only.** There is no SMB in it. SMB is new work. libsmb2 (same author as libnfs, LGPL-2.1) has a similar synchronous POSIX-like API and a `PS4_PLATFORM` build path. See [swanstation-nfs-reference.md](./swanstation-nfs-reference.md).
2. **ROMs are tiny.** The largest ROM is 16 MiB (`MAX_ROM_SIZE`) and the core copies it into RAM at launch. So smoothness comes from design: fetch the whole file once, then the network is idle during play. SwanStationPS5's 128 MiB streaming cache exists for PS1 images of hundreds of MB. Here only MSU-1 audio needs a streaming cache.
3. **Scanning is the slow part**, not launching. Today each file costs one `stat`, and each unknown CRC reads the whole ROM. Over a network that is thousands of round trips and gigabytes.
4. **Sibling-file probing** (patches, MSU-1) means a dozen failed opens next to every ROM at load.
5. **The console has quirks** (resolver, `fcntl` on sockets, small heap) that must be proven before any design is trusted.

## Smooth playback contract

These are the acceptance rules. Each has an automated check on the host unless noted.

| # | Rule | Check |
|---|---|---|
| 1 | While a game runs, a read on the emulation thread waits at most 4 ms, then returns silence. An MSU-1 open is served from a prefetch; with no prefetch it waits one round trip and logs it. | Fake backend with 200 ms latency and hangs; loop-timing assertions. |
| 2 | After load, a game without MSU-1 causes zero network reads. | Backend read counter stays 0 over 600 frames. |
| 3 | Launch reads the ROM once, in order, and nothing else beyond the folder listing. Target: first frame under 1 s on gigabit for a 6 MiB ROM. | Backend bytes read equal the file size; time measured on console (unverified). |
| 4 | Losing the connection never ends a running game. ROM games continue; MSU-1 tracks go silent with a notice. | Kill the fake backend mid-game. |
| 5 | An MSU-1 underrun yields silence for the missing span and keeps time aligned. Underruns are logged. | Fake backend with 200 ms latency. |
| 6 | The shelf opens at once. Local games first; network games join when the share is listed; the selection stays. | Host test with a share that never answers. |
| 7 | Saves, states and cheats stay local. An offline share never delays saving. | Existing save tests, share offline. |

## Design

```
 fe_games / fe_shelf / fe_menu            Snes9x core, minizip, zlib (unchanged)
        |  mount points, status                    |  fopen gzopen unzOpen stat opendir ...
        v                                          v
   netfs.h (C API) ----------------->  linker --wrap  ->  netfs_vfs   /smb/<host>/<share>/...
                                                                       /nfs/<host>/<path>
                                                               |
                          netfs_share (config, state)          +--> netfs_cache
                          netfs_watchdog (5 s kill)                 whole-file fetch
                                                                    read-ahead (big files)
                                                                    dir + attr + miss cache
                                                               |
                                          backend_smb (libsmb2)   backend_nfs (libnfs)
```

- **netfs_vfs** answers every libc file call for virtual paths and forwards the rest to the real call. The core never knows.
- **netfs_cache** decides how bytes reach the reader (spec section "Reads").
- **backend** is a small interface: connect, stat, list (names and attributes together), open, pread, close, abort. One lock per connection because both libraries are not thread safe.
- **netfs_share** parses addresses, mounts in the background, keeps per-share state and plain-word errors.
- **net shims** (PS5 build only) give libsmb2 and libnfs a working resolver and non-blocking sockets.

## Phases

Sizes are relative effort, not time: S small, M medium, L large.

| Phase | Deliverable | Exit criteria | Size |
|---|---|---|---|
| 0. Spikes | Throwaway branch answering six yes/no questions with numbers (spec, "Spikes"). | Results written to this folder. If libsmb2 fails on the console, Phase 1 uses NFS first. | M |
| 1. netfs core + SMB | `ps5/netfs/`: vfs and wrap list, whole-file fetch, attribute cache, share state, watchdog, SMB backend, fake backend. Shares from `shares.txt`. Mount points added to `OrbisRomRoots()`. | Host tests pass. On the console a ROM from an SMB share plays and rules 1-4 hold. | L |
| 2. Library and UI | Two-pass scan, background mount, rescan, NET badge, root labels, error notices, Network status screen, deferred CRC. | Shelf never blocks. 2,000 ROMs scan in about one round trip per folder on the fake backend. | M |
| 3. NFS | NFS backend on vendored libnfs with the PS5 config. Export listing in errors. | Same tests run over both backends. | M |
| 4. MSU-1 | Read-ahead ring for big files, track-head prefetch, silence on underrun. | Rule 5 holds on the fake backend. Console test with an MSU-1 hack. | M |
| 5. Polish | README section and credits. On-screen keyboard to add shares. Optional local ROM cache. Automatic retry. | Jason adds a share without FTP. | M |

Each phase is its own PR to `snes9xPS5`. Phase 0 is not merged.

## Decisions (defaults chosen; change them here)

| # | Decision | Default | Alternative |
|---|---|---|---|
| D1 | Backend order | SMB first (your stated need), NFS second. Swap if the libsmb2 spike fails. | NFS first: proven on this toolchain by SwanStationPS5. |
| D2 | Reuse SwanStationPS5 code? | No. Reimplement from the design (GPL vs MIT, ADR 001). Vendor only libsmb2 and libnfs. | Ask darkxex for permission to adapt `nfs.cpp`. |
| D3 | How shares are added | `/data/snes9x/shares.txt` plus a read-only Network screen. Keyboard in Phase 5. | Build the keyboard first. |
| D4 | Passwords | Plain text in `shares.txt` (the console has no secret store); redacted in logs and screens. Guest access works with none. | Ask at connect time (needs the keyboard). |
| D5 | Local ROM cache | Off. Fetch into memory at every launch. | LRU folder under `/data/snes9x/netcache`, capped. |
| D6 | Server names | IPs and DNS names. NetBIOS names not supported; docs say use the IP. | Add NetBIOS or mDNS lookup later. |

## Risks that can stop the plan

| Risk | Why it matters | Answer |
|---|---|---|
| libsmb2 does not build or run on the console | SMB is the headline. | Spike S3-S4. Fall back to NFS first (D1). |
| Router USB shares often speak SMB1 only | libsmb2 is SMB2/3. | The error says "server only speaks SMB1"; docs list Windows, Samba and NAS setups. |
| Licences | libsmb2 and libnfs are LGPL-2.1; Snes9x's own licence is non-commercial; the SDK CRT is GPL. | Jason confirms before Phase 1 ships. Not legal advice. |
| No console in the cloud | Rules 3 and 5 need hardware numbers. | CI uploads the ELF; the spec has a hardware checklist for Jason. |

More risks and the test plan are in the spec.
