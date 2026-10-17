# Findings: what Skyline, ARCropolis, Eden and ProsperoEden need from each other

Phase 0 output, 2026-10-17. Every answer cites the file and line it comes from. Later phases cite this file instead of guessing. Where something could not be read in source it says **not determinable**, with the run that settles it (those runs are in [hardware-test-protocol.md](./hardware-test-protocol.md)).

## Sources read

| Source | Version | Notes |
|---|---|---|
| ProsperoEden `main` | `23e0424` | Paths below are in this repo unless prefixed. |
| Eden (pin in `UPSTREAM.json`) | `67bada77` | Fetched with `git fetch` of that commit from `eden-emulator/mirror`. Prefix `eden:`. |
| Skyline (`skyline-dev/skyline`) | `2eb226f`, 2026-04-12 | Prefix `skyline:`. |
| ARCropolis (`Raytwo/ARCropolis`) | `bf2005b`, 2026-09-26, crate version 4.1.1 | Prefix `arc:`. Release list read on the GitHub releases page. |
| Ryujinx (Ryubing fork) | `a82350b`, 2026-07-17, mirror at `codeberg.org/Software_arch/Ryubing` | Prefix `ryu:`. The upstream repositories (`Ryujinx/Ryujinx`, `Ryubing/Ryujinx`) and every yuzu repository are not reachable from the cloud container. |

Labels: **[src]** read in the files named. **[web]** read on a web page. **[inferred]** follows from [src] but no line says it. **[unverified]** not checked.

## Summary

| # | Question | Short answer | Gap | Phase |
|---|---|---|---|---|
| 1 | Smash version, title IDs | ARCropolis 4.1.1 runs on **13.0.5 only**. Base `01006A800016E000`, update `01006A800016E800`, DLC `01006A800016F000`. | The update's display version must read exactly `13.0.5`; nothing in ProsperoEden checks it. | 5, 8 |
| 2 | Files installed | Skyline: `exefs/subsdk9` + Smash's `exefs/main.npdm` + plugins in `romfs/skyline/plugins/`. ARCropolis: `libarcropolis.nro` there, everything else under `sd:/ultimate/`. The hashes file is optional. | Plan Background items 2 and 3 were partly wrong (fixed below). | 8 |
| 3 | `main.npdm` and `subsdk9` from a mod | **Yes, already works** in Eden's ExeFS layering, and `roms/<Game>/mods/<mod>/exefs/` reaches it. | Layer order is by mod folder name; unverified on console. | 3 |
| 4 | Services and SVCs | Every SVC and service Skyline and ARCropolis call at start is present in Eden. None are stubbed. | Sockets (log port 6969) and `ninput` are the soft spots. | 3, 5, 6 |
| 5 | SD card | Eden already mounts one at `/data/prosperoeden/user/sdmc` and layers `atmosphere/contents/<title>/` from it. Undocumented. | Location (D1), lazy creation, case sensitivity, free-space walk. | 2 |
| 6 | Self-modifying code | Eden invalidates the JIT on `ic ivau` / `ic iallu`; ProsperoEden's shared JIT carries that to every core. | Only the console can prove it, with Compile ahead on and off. | 3, 7 |
| 7 | `make test` | **Does not run in the cloud container**: the dependency downloads are blocked. | Host checks cannot be run by cloud threads. | 1 |
| 8 | Memory | Memory layout is 4/6/8 GiB per game, default 4. Smash's and ARCropolis's sizes are **not determinable** from source. | Needs a measurement. | 7 |
| 9 | Load address (found while answering 3 and 6) | ARCropolis calls the console "an emulator" only when the main module's text starts at `0x8004000` or `0x8504000`. Eden loads at `0x80000000`+. ProsperoEden's seed 0 does **not** give either. | Only `ninput::init()` depends on it in ARCropolis itself. | 3, 5 |

## 1. Smash version and title IDs

**Answer.** ARCropolis 4.1.1 (the newest release, 25 Sep) runs only on Smash **13.0.5**: at start it reads the game's display version and stops with "ARCropolis cannot currently run on a Smash version other than 13.0.5" for any other [src arc:`src/lib.rs:516-520`]. Release 4.0.9 added "Support for update 13.0.5" [web: GitHub releases page]. D3 is therefore 13.0.5, not "the newest the player has".

Title IDs: base `01006A800016E000` [src arc:`Cargo.toml:8`, skyline:`cross.json:3`]. Eden derives the update as base `| 0x800` = `01006A800016E800` [src eden:`src/core/file_sys/registered_cache.h:63-65`] and the add-on content base as base `+ 0x1000` = `01006A800016F000` [src eden:`src/core/file_sys/common_funcs.h:11,47`]. The DLC pack's individual IDs run upward from that base [inferred].

Stale comments inside ARCropolis disagree with the check (`13.0.2` in `src/fixes.rs:52`, `13.0.4` in `crates/arcadia/tools/addresses.toml:1`, `13.0.5` in `crates/arcadia/src/offsets.rs:2`). The check in `lib.rs` decides.

**How the version reaches ARCropolis.** It calls `nn::oe::GetDisplayVersion` [src arc:`src/utils.rs:37-38`]. Eden answers it from the game's control data (NACP) and falls back to `1.0.0` when there is none [src eden:`src/core/hle/service/am/service/application_functions.cpp:217-232`]. A game started from NSP/XCI with its update beside it (docs/INSTALL.md) should report the update's version; whether ProsperoEden's loader picks the update's NACP, and not the base game's, is **unverified**.

**Gap.** No check or message in ProsperoEden when the version is not 13.0.5. **Phase 5** runs the version test (ARCropolis's own dialog is the message); **Phase 8** states 13.0.5 in the guide.

## 2. What gets installed, and where

**Skyline** [src skyline:`README.md:6-10`, `scripts/sendPatch.py:70-83`]:

| File | Role |
|---|---|
| `exefs/subsdk9` | Skyline itself, an NSO with a `MOD0` header the game's own rtld loads after `main` (`source/crt0.s`). |
| `exefs/main.npdm` | A replacement process descriptor. The shipped one is for Smash (`cross.json`); the README says to delete it for any other game. |
| `romfs/skyline/plugins/*.nro` | Plugins. Skyline reads them from the game's own mounted romfs (`PLUGIN_PATH = "skyline/plugins"`, `include/skyline/plugin/PluginManager.hpp:13`; mount name taken from the game's `MountRom` call, `source/main.cpp:38-41`). |

What `main.npdm` changes against Smash's own [src skyline:`cross.json`]: every SVC allowed (0x00 to 0x7f, including `MapProcessMemory` 0x74 and `UnmapProcessMemory` 0x75), `service_access: ["*"]`, filesystem permissions all set, `allow_debug: true`, handle table 512, main thread stack 1 MiB, 39-bit address space (`address_space_type: 3`), application pool, `min_kernel_version 0x0091`, title ID range wide open. Eden enforces the SVC list from the process's `main.npdm` (`IsPermittedSvc`, [src eden:`src/core/hle/kernel/k_capabilities.h:50-52`, `k_process.h:276`]), so Skyline needs its own `main.npdm`, and the stock Smash one lacks the SVCs Skyline calls **[inferred: not compared byte for byte]**.

**ARCropolis** [src arc]:

| Path | Role | Required |
|---|---|---|
| `romfs/skyline/plugins/libarcropolis.nro` | The plugin (`.github/workflows/build.yml:48-53`). | yes |
| `sd:/ultimate/mods/<mod>/` | ARC mods (`src/utils.rs:61`). | the mods |
| `sd:/ultimate/arcropolis/config/<uid>/<uid>/` | Settings, per user: the user is the game's preselected user (`crates/config/src/lib.rs:303-325`, `.expect` on failure). Created on first run. | created by it |
| `sd:/ultimate/arcropolis/{logs,cache/<version>}` | Log and discovery cache (`src/utils.rs:69-79`). | created by it |
| `sd:/ultimate/arcropolis/resources/` | Files for its in-game menus. Missing ones switch the menus off for that boot and show an error (`src/lib.rs:602-618`); the game still starts. | no |
| `sd:/ultimate/arcropolis/hashes.txt` | **Optional.** Read only to name files in log lines: "There won't be any hash lookups in this run's logs" (`src/hashes.rs:5-27`). The plan assumed ARCropolis needs it to map file names. It does not. | no |
| `sd:/ultimate/arcropolis/conflicts.json` | Written when two mods claim one file (`src/fs/discover.rs:269-277`). | output |

The ARC mods themselves are walked with ordinary `read_dir` and `metadata` on `sd:/ultimate/mods` (`src/fs/discover.rs:70-96,203`), including a second pass for fighter `.nro` modules under `prebuilt;/nro/release` that it registers with `ro` (`discover.rs:318-340`). The CI package is built with `--no-default-features` (no menus, no updater); the default features are `ui` and `online` (`Cargo.toml:72`), so a release the player downloads likely has both **[unverified]**: the asset names could not be read.

**Updater and network.** With `online`, ARCropolis checks for a newer release over HTTPS at start unless `auto_update` is off; the default config sets it **on** (`crates/config/src/lib.rs:53,64`). It would try to download and replace its own `.nro`. The guide has to tell the player to turn it off (Phase 8), and Phase 5 tests that it fails fast without a network.

**Gap.** Layout for the guide; no ProsperoEden code. **Phase 8** (guide), **Phase 5** (offline test).

## 3. Does Eden's ExeFS patching honour `main.npdm` and `subsdk9` from a mod?

**Answer: yes.** [src]

1. `AppLoader_DeconstructedRomDirectory::Load` reads `main.npdm`, then, when constructed with `override_update`, calls `PatchManager::PatchExeFS` and **re-reads `main.npdm` afterwards** "in case PatchExeFS affected the main.npdm" (`eden:src/core/loader/deconstructed_rom_directory.cpp:157-179`). The loop over `{"rtld","main","subsdk0".."subsdk9","sdk"}` then reads each module from that patched directory (`:191-212, 248-262`).
2. `PatchExeFS` layers the `exefs` folder of every mod (in the order of the mod folders' names, skipping the switched-off ones) over the game's ExeFS; the first layer holding a file wins (`eden:src/core/file_sys/patch_manager.cpp:331-359`). Mods come from `<load dir>/<title>/<mod>/exefs` and from `sdmc/atmosphere/contents/<title>/exefs` (`:336-340`; the SD folder is the layer named `SDMC`).
3. ProsperoEden hands Eden the game's mods through the derived `bis_factory.cpp`: `GameFolderMods::Root` returns the layered folder of `roms/<Game>/mods/` plus `mods/<title ID>/` [src `headless/game_folder_vfs.h:21-48`, `headless/CMakeLists.txt:~1590-1610`, `headless/mods.h:1-37`]. `Mods::Kinds` already counts `main`, `main.npdm`, `rtld`, `sdk` and `subsdk0..9` as code (`mods.h`, `kCodeFiles`).

So `roms/Smash/mods/Skyline/exefs/{subsdk9,main.npdm}` and `roms/Smash/mods/Skyline/romfs/skyline/plugins/libarcropolis.nro` are the whole install through the mods path. Plugins ride on LayeredFS: a `romfs` folder in a mod is merged into the game's romfs, new files included, and the romfs image is rebuilt from the layers (`eden:patch_manager.cpp:531-600`). Rule 2 of the README stands; ADR 001 stands and is now Accepted.

**Things to know.**
- The process is built from the re-read `main.npdm`: its title ID, address space, pool, thread stack and SVC mask are the mod's (`eden:src/core/hle/kernel/k_process.cpp:1191-1259`). Two mods that both ship a `main.npdm` or `subsdk9`: the one whose folder name sorts first wins, silently.
- The patch step runs only when the loader is built with `override_update` (`deconstructed_rom_directory.cpp:167`). It is **true** for an NSP or XCI game, which loads through its program NCA (`eden:src/core/loader/nca.cpp:91`), and **false** for a game that is a bare `exefs` folder or an extracted NSP (`loader.cpp:345`, `nsp.cpp:38`). The docs say games are NSP or XCI, so the mods path applies. A game stored as an extracted folder would load `main` but get no mods at all.
- `nca.cpp:92-110` reads the heap size and so the IPC pointer buffer size from the **unpatched** `main.npdm`, before the mods are layered. Skyline's npdm does not change what that code reads **[inferred]**; a mod `main.npdm` with another heap size would be ignored there.
- **Phase 3** checks the layering in the log first (Eden prints "ExeFS: LayeredExeFS patches applied successfully", `patch_manager.cpp:358`, and `metadata.Print()` after the re-read).
- ProsperoEden changed this area: `sdmc_load_dir` and a missing mod root are handled in `headless/CMakeLists.txt:1600-1604`.

**Gap.** Verify on the console; document layer order. **Phase 3**, **Phase 8**.

## 4. Services and SVCs Skyline and ARCropolis use at start

### SVCs

| SVC | Used by | Eden at the pin |
|---|---|---|
| `GetInfo` 65001 `MesosphereCurrentProcess` (own process handle) | Skyline `cur_proc_handle.cpp:63` | **Implemented**, returns a new handle to the current process (`eden:svc/svc_info.cpp:241-258`). |
| `CreateSession` 0x40, `CreateThread`, `StartThread`, `SendSyncRequest` with a copied process handle, `ReplyAndReceive` 0x43, `WaitSynchronization`, `ExitThread` | Skyline's fallback if 65001 fails (`cur_proc_handle.cpp:30-60`) | Present; unreachable while 65001 works. |
| `MapProcessMemory` 0x74, `UnmapProcessMemory` 0x75 | Skyline, every inline hook and code patch: map the code page read-write elsewhere, write, `ic ivau`, unmap (`inlinehook/controlledpages.cpp:120-153`) | **Implemented** (`eden:svc/svc_process_memory.cpp:62-123`). The `Code` memory state has `FlagCanMapProcess` (`eden:k_memory_block.h:64`), as hardware does. |
| `QueryMemory`, `GetInfo` (region and program ID infos) | Skyline `utils.c`, `controlledpages.cpp:57-64` | Standard. |
| `OutputDebugString` 0x27 | Skyline's logger (`DualLogger.cpp:114`) | Implemented; logs at **Info** in class `Debug_Emulated` (`eden:svc/svc_debug_string.cpp:25`), so Skyline's lines reach the ProsperoEden log. |
| `FlushProcessDataCache`, `InvalidateProcessDataCache` | none seen | Present. |

### Services (called through the game's own nnSdk)

| Service | Used by | Eden |
|---|---|---|
| `fsp-srv`: `OpenSdCardFileSystem` (cmd 18), file and directory calls | Skyline mounts `sd` with `nn::fs::MountSdCardForDebug("sd")` (`skyline:source/main.cpp:106`); ARCropolis reads everything under `sd:/` | Implemented (`eden:.../fsp/fsp_srv.cpp:65,224`). Returns `ResultPortSdCardNoDevice` if there is no SD factory (`filesystem.cpp:396-410`); there always is. |
| `fsp-srv` save data | ARCropolis mounts `save:` for the region (`utils.rs:122-150`) | Standard save mount **[unverified for the preselected user form]** |
| `ldr:ro` (`Initialize`, `RegisterModuleInfo`, `LoadModule` via `MapManualLoadModuleMemory`) | Skyline plugin loader (`PluginManager.cpp:18,143,165`), ARCropolis chain loader and fighter NRRs | **Implemented** from Atmosphere's ro, up to 256 NRRs and 256 NROs, SHA-256 checks against the NRR (`eden:src/core/hle/service/ro/ro.cpp:33-34,196-210,544-550`). More headroom than hardware. |
| `bsd:u` (`RegisterClient`, Socket, Bind, Listen, Poll, Accept, Send) | Skyline's log server: TCP **port 6969 on any address**, accepts one client, no authentication (`DualLogger.cpp:7,20-66`). The game's later `nn::socket::Initialize/Finalize` are hooked to do nothing (`:94-105`). | Implemented; ProsperoEden's own BSD layer (`headless/multiplayer_bsd.inc`) and the `airplane_mode` setting, which makes TCP socket creation fail with `NOTCONN` (`:12-13`). Skyline's logger returns quietly on any socket failure (`DualLogger.cpp:23,35,41,62`). |
| `acc:u0`, `am` launch parameter with the preselected user | ARCropolis config and save mount | Eden pushes an `AccountPreselectedUser` launch parameter (`eden:.../am/applet_manager.cpp:27-35,298-314`). |
| `am` `GetDisplayVersion` | ARCropolis version check | Implemented, see question 1. |
| `time:u` (`nn::time::Initialize`) | ARCropolis log file names (`lib.rs:205-211`) | **Unverified**; Phase 3 reads the log. |
| `hid` | `ninput::init()` runs in ARCropolis **unless** it believes it is on an emulator (`lib.rs:528-531`) | Eden's hid is HLE; a direct `ninput` use is **unverified**. See question 9. |
| `nifm`, `ssl`, `bsd` | ARCropolis's updater (`online` feature) | Present in Eden; behaviour with no route **unverified**. |

**Gap.** None is missing. The unverified ones are exactly what Phase 3 and Phase 5 find out from the first logs. Skyline's TCP listener is an open, unauthenticated port on the console: Phase 6 decides whether to leave it, and the guide says so.

## 5. SD card

**Answer.** Eden already mounts an SD card and ProsperoEden already places it: `/data/prosperoeden/user/sdmc`. [src]

- Eden's path manager gives `SDMCDir = <Eden user path>/sdmc` (`eden:src/common/fs/path_util.cpp:168`, `fs_paths.h:29`); ProsperoEden sets the Eden user path to `Eden::UserDir()` = `/data/prosperoeden/user` (`headless/CMakeLists.txt:2851`, `headless/storage_paths.h:78`).
- `CreateFactories` opens it read-write (`eden:filesystem.cpp:729-735`) and also `sdmc/atmosphere/contents` as the SD layer of mods (`:730`). So a player can install Skyline the Atmosphere way, into `sdmc/atmosphere/contents/01006A800016E000/{exefs,romfs}`, with no change at all; `roms/<Game>/mods/` is the same thing per game.
- `docs/INSTALL.md` does not mention `user/sdmc`. Nothing in ProsperoEden creates or lists it.

**Questions the plan asked.**

| Question | Answer |
|---|---|
| Case sensitivity | The console's host file system is case-sensitive; the Switch's SD card (FAT) is not. Eden has no case folding for guest paths (only `FindSubdirectoryCaseless` for the mod folders). ARCropolis lowercases only the game's own paths (`lib.rs:134`) and walks mod folders as found, so a mod folder written in capitals works on a Switch and is **not determinable** here. Ryujinx's SD card uses the host's case rules too (`ryu:src/Ryujinx.HLE/FileSystem/VirtualFileSystem.cs:190`). Test: protocol "case" run. |
| Directory listing | ARCropolis lists every mod folder recursively at start and caches the result (`discover.rs:228-232`). ProsperoEden needs `ReadNativeDirectory` on the console (`mods.h:102-117`): Eden's own directory iterator does not work there. Whether Eden's SD directory code uses it is **unverified**; the first run shows it. |
| Big files | Reads go through the same file code as game files, which patch 01 sped up (`tools/eden-patches/01-fast-file-reads.diff`); ARCropolis's `mods:/` accessor reads a file whole on every call (`ModFileAccessor::read`, `src/fuse/mods.rs`). Performance is **not determinable**; Phase 5 and 7. |
| Free space | `SDMCFactory::GetSDMCFreeSpace` calls `sd_dir->GetSize()`, which **walks the whole SD folder** (`eden:src/core/file_sys/sdmc_factory.cpp:57-59`). Called when a game asks `nn::fs::GetFreeSpaceSize` or similar; with many gigabytes of mods that is slow. Nothing in the two programs' start paths calls it **[unverified]**. |
| Creation | The folder is created by Eden's path manager at start; `mods.h` says the app never creates a game folder behind the player's back. Whether that rule covers `user/` (which already exists for saves) is a Phase 2 decision. |

**D1 (location).** The folder exists, so the recommendation changes: keep `/data/prosperoeden/user/sdmc` as the SD card (it already holds Eden's `Nintendo/Contents` and album), and document it. A card that travels with the games on USB/M.2 (`<game files folder>/sd`) is still possible as a setting. Phase 2 asks Jason.

**Gap.** Document it; decide D1; case handling; free-space walk. **Phase 2**.

## 6. Self-modifying code and the JIT

**Answer.** The mechanism Skyline uses is handled. [src]

1. Skyline writes through a read-write alias (`MapProcessMemory`), flushes with `dc civac` on the alias and `ic ivau` on the original per cache line, then unmaps (`controlledpages.cpp:145-153`, `nx/arm/cache.s:22-67`).
2. Eden's A64 JIT raises `InstructionCacheOperationRaised`; for `ic ivau` it invalidates the 64-byte line (`CTR_EL0` line size assumed, `eden:src/core/arm/dynarmic/arm_dynarmic_64.cpp:90-104`), for `ic iallu` the whole cache, and halts the core so it re-dispatches. `ic ialluis` is **not** handled (logged only, `:101-103`). Skyline uses `ic ivau` only.
3. ProsperoEden shares compiled blocks across the process's cores (`headless/dynarmic/jit_group.h:1-20`). Invalidation takes the group's lock, drops every block overlapping the range, unlinks the branch sites that jumped to them and halts the other cores so they flush their dispatch tables (`headless/dynarmic/jit_impl.inc:100-107, 365-399`; `jit_group.h:152-166`). A core mid-block keeps running the old code until it next reaches the dispatcher, the same as hardware without a barrier.
4. Kernel paths that change code also invalidate: `UnmapCodeMemory`, `SetProcessMemoryPermission`, `WriteDebugMemory` (`eden:k_page_table_base.cpp:1345, 2083, 3328`).
5. **Compile ahead** (`headless/jit_list.h`) records block locations of the modules mapped at start, per game build, and compiles them on a spare CPU while the game starts. It is safe with hooks by design: such blocks are "invalidated like any other", and a precompiled block is dropped if any invalidation happened while it compiled (`jit_list.h:22`, `jit_group.h:168-186`). The lists saved by earlier sessions name the post-hook locations of Skyline's own module too, which is harmless. This is design reading only.
6. The JIT's page table and "page reuse" (`headless/jit-page-table.cmake`, `dynarmic/page-reuse.cmake`) cache a guest page's host pointer; an alias page created by `MapProcessMemory` goes through the normal map path, so it gets its own entry. **Unverified.**

**Gap.** No code gap found. Console run with Skyline and ARCropolis installed, once with Compile ahead off and once on, then compare. Phase 3 runs the first; the second is in Phase 7. No escalation to Opus was needed: one read of `arm_dynarmic_64.cpp` and the JIT files was enough.

## 7. Does the container run `make test`?

**No.** [src, tried]

- `make test` first runs `deps` (`Makefile:57`), which downloads about fifteen inputs. The proxy answers **403** for `github.com/.../archive/...` and `.../releases/download/...` and for `codeload.github.com` (tried `python3 tools/deps.py fetch fmt`: three attempts, HTTP 403). Git over HTTPS and `raw.githubusercontent.com` work, which is how this phase got Eden, Skyline and ARCropolis.
- Even with the downloads, the host build also needs ffmpeg and a C++ dependency tree built from source, and `clang-18` (`tools/build-headless-host.sh`). `clang-18` is installed. The sanitizer checks (`check-heap-growth.py`, `check-stop-limit.py`) fail at link time with `-fsanitize`.
- These ran: `python3 tools/check-guest-clock.py` (pass), `check-gc-scan.py` (structural checks pass, Eden comparison skipped).

**Consequence for every code phase.** The cloud thread cannot run the host suite. It says so plainly in its PR ("host tests not run: container cannot download dependencies") and Jason or CI runs `make test`. To change this, the project's cloud environment needs `github.com` allowed in its network settings (Project settings > environment); a decision for Jason, not for a phase.

**Gap.** **Phase 1** records this in the baseline and asks Jason once whether to widen the network access.

## 8. Memory

**Answer.** Partly determinable.

- ProsperoEden gives a game 4, 6 or 8 GiB of console memory, per game or for all, default 4 (`headless/main.cpp:1274-1280`, `headless/preferences_check.cpp:369-386`, docs/SETTINGS.md "Console memory").
- `docs/MODS.md` already warns that some mods read the kernel's memory figure and take a block out of it, and fail on 4 GiB.
- Skyline's `main.npdm` sets no memory size of its own; it is the process descriptor of Smash [src `cross.json`].
- **Not determinable from source:** Smash's resident size (the game's NACP and `data.arc` mapping are not in the sources read), ARCropolis's resident size (the discovery tree, all hashes and the decoded resources are held in memory, the tree grows with the number of mod files), and the size of the 6969 log buffer. Needs the baseline (Phase 1) and one run with mods (Phase 7).

**Gap.** **Phase 7** measures; **Phase 3** records Skyline's own cost in the first log.

## 9. Load address: ARCropolis's emulator check (found, not asked)

**Answer.** ARCropolis decides "emulator or console" from where the main module's text begins: `0x8004000` or `0x8504000` is an emulator, anything else a console (`arc:src/utils.rs:14-27`). Skyline's `getRegionAddress(Text)` is the text map that contains `nninitStartup`, that is the **main** module (`skyline:source/skyline/utils/cpputils.cpp:19-22,181-185`).

Where the two emulators put it:
- **Ryujinx**: the process starts at `0x8000000 + 0x500000` = `0x8500000` (`ryu:src/Ryujinx.HLE/Loaders/Processes/ProcessLoaderHelper.cs:33,251`); `rtld` is the first module and `main` follows it, which gives `0x8504000` if `rtld` is one `0x4000` block **[inferred]**.
- **yuzu, before it was stopped**: `0x8000000`, hence `0x8004000` **[inferred from the constant]**.
- **Eden at the pin**: a 39-bit game starts at `0x8000'0000 + aslr_offset`, "about 2GiB into the ASLR region", because "some (buggy) programs/libraries like skyline incorrectly depend on the existence of ASLR pages before the entry point" (`eden:src/core/hle/kernel/k_process.cpp:1221-1226`). Eden's own comment names Skyline.
- **ProsperoEden**: the offset is random per start, or fixed by the game's RNG seed (`headless/load_address.h`, `headless/CMakeLists.txt:1286`). The comment in `load_address.h` says mods that tell an emulator by the address "ask for seed 0, which is no offset at all". Seed 0 gives `0x80000000`, not `0x8000000` or `0x8500000`: **it does not satisfy ARCropolis's check**.

**What the check changes in ARCropolis.** One thing: on a "console" it runs `ninput::init()` (`arc:src/lib.rs:528-531`), the crate that reads the pad directly through the game's `nn::hid`. Other plugins (HDR and others) may check too. **Unverified:** whether `ninput::init()` works on Eden's HLE hid or hangs.

**Options.** (a) Do nothing and see if `ninput::init()` works (cheapest; Phase 5 finds out). (b) A numbered patch that loads 39-bit games at `0x8000000` for a game with Skyline, which Eden's comment says breaks Skyline's own assumptions, so only if (a) fails. Not decided here.

**Gap.** Skyline's dependence on pages before the entry point is a Phase 3 risk; the `ninput` question is Phase 5. The comment in `load_address.h` is wrong about what seed 0 does: fix it in **Phase 3** or **Phase 5**, whichever touches the file.

## Ryujinx and yuzu

Jason asked that both be studied: [emulator-reference.md](./emulator-reference.md). Short form: Ryujinx has no Skyline-specific code, only the generic loader and kernel features listed here in questions 3, 4 and 6; Eden (yuzu's descendant) has them all, and its source even names Skyline; no yuzu repository could be reached.

## Plan corrections

The plan's Background, D1 and D3, the README's rules 3 and 8, ADR 001 and the phase scopes were corrected in the same change as this file; the Background list in the plan has the short form.

## Console runs added to the protocol

From the **not determinable** and **unverified** items: version string on the screen; `override_update`/`main.npdm` in the log; `ninput::init()` outcome; `time`/`nifm` behaviour; case of mod folder names; first-start time with a large `ultimate/mods`; log line count with a dev build. They are in [hardware-test-protocol.md](./hardware-test-protocol.md).
