# How Ryujinx and yuzu ran Skyline and ARCropolis

Split from [findings.md](./findings.md) (phase 0). Same source labels and versions as there.

Jason: "Yuzu and Ryujinx were able to do it in the past. Look at both." What could be read:

**Ryujinx (Ryubing fork, source read).** It contains no Skyline-specific code at all (`grep -ri "skyline\|arcropolis"` finds only the NSO name list). It runs Skyline because its generic loader and kernel already did what Skyline needs, which is the same list as here:

| Need | Ryujinx | Eden at the pin | ProsperoEden |
|---|---|---|---|
| `subsdk0..9` and `sdk` loaded after `main`, in order | `Loaders/Processes/ProcessConst.cs:7-21` | same list, `deconstructed_rom_directory.cpp:191` | same (Eden's) |
| Replacement NSOs and `main.npdm` from a mod's `exefs` folder; a mod can also ship a whole `exefs.nsp` | `HOS/ModLoader.cs:572-690` (`ApplyExefsMods`: replaces NSOs by name, "main.npdm replaced", `.stub` files drop an NSO) | layered ExeFS, see 3 (no `.stub`, no whole-partition replacement) | `roms/<Game>/mods/<mod>/exefs/` |
| SD card with Atmosphere layout | `sdcard/` folder; mods from `sdcard/atmosphere/contents/<title>` (`Ryujinx.Common/Configuration/AppDataManager.cs:37,326`) | `user/sdmc`, `atmosphere/contents` layer | same, undocumented |
| `GetInfo` 65001 | `Kernel/SupervisorCall/InfoType.cs:35`, `Syscall.cs:2205` | `svc_info.cpp:241` | same |
| `MapProcessMemory` | `Syscall.cs:1570` | `svc_process_memory.cpp:62` | same |
| `ic ivau` invalidates translated code | `ARMeilleure/Instructions/NativeInterface.cs:192-195` | `arm_dynarmic_64.cpp:90` | plus group-wide invalidation |
| 39-bit code start | `0x8500000` | `0x80000000`+offset | seed or random |
| SD card path case rules | host default (`LocalFileSystem.PathMode.DefaultCaseSensitivity`) | host | host |

The ARCropolis side confirms the picture. Its README says "only Ryujinx has some compatibility ... support for Yuzu is being worked on" [src `arc:README.md`], and the maintainers state they will not support emulators, will not ship emulator-specific builds, and that "future updates may break" it (they name a custom exception handler and a custom `nn::fs` implementation as planned) [web: ARCropolis issue #195]. So **expect to follow ARCropolis releases**: 4.1.1 works only with 13.0.5 and may use anything the Switch has.

**yuzu.** No yuzu repository could be reached (shut down 2024; the forks sampled are not on a host the cloud container may use). Eden is yuzu's descendant and carries the same kernel and service code, so Eden's source is the yuzu reference: its comment about Skyline (`k_process.cpp:1221-1225`), `MesosphereCurrentProcess` (`svc_info.cpp:241`), `MapProcessMemory` (`svc_process_memory.cpp`), the SD layer of mods (`filesystem.cpp:730`) and the `ro` service all exist for this. A GameBanana question in 2026 reported ARCropolis 4.0.8 failing on yuzu with a file system mount error while "working fine" on Eden [web: search result only, the page's replies could not be read; **treat as unconfirmed**].

**What this tells the plan.** (1) The emulator side is mostly done; the work is in ProsperoEden's own wiring (SD card folder, versions, logs, launcher) and in finding what is different on the PS5 (the JIT, memory, the file system). (2) The load address is the one place Ryujinx's and Eden's layouts differ in a way ARCropolis notices.

