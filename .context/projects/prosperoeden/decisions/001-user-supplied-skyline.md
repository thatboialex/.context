# ADR 001: Skyline and ARCropolis are user-supplied mods, not bundled

Status: Proposed, 2026-10-17.

## Context

ProsperoEden already loads exefs and romfs mods from `roms/<Game>/mods/` and states that it includes and downloads no mods. Skyline is delivered as exefs files (`subsdk9`, `main.npdm`) and romfs plugins; ARCropolis is a plugin that reads an SD card folder.

## Decision

Treat Skyline plus ARCropolis as an ordinary ProsperoEden mod the player copies in, and add only what the emulator needs to let it run: an SD card folder, any Eden fixes it exposes, a launcher status row, diagnostics. No bundling, no downloading, no in-app installer.

## Consequences

- No licence or redistribution questions, and no release depends on their versions.
- Setup is manual; the guide (phase 8) must make it short and exact.
- Anything that needs the mod's contents (hashes, plugin versions) is documented, not shipped.
- If phase 0 shows the mods path cannot carry Skyline, supersede this ADR before building a new mechanism.
