# ADR-004: Shelf tabs (Internal, USB, Network) and a PSFlycast-style Network tab

## Status
Proposed, 2026-10-17 (snes9xPS5 draft PR, host tests pass). Becomes Accepted when Jason has used it on the console. Unverified on hardware: the tab bar at 1080p, the pad feel and the Network tab's states against a real share have only been seen in host renders.

## Context

Jason played from a share on the console (the first real test of netfs) and asked for the network part to follow [PSFlycast](https://github.com/Press5elect/PSFlycast) and for games on the console's drive to be shown apart from games on a share. Until now the shelf is one flat list: network games sit among the local ones with a `NET` badge.

What PSFlycast does (its `shell/ps5/README.md` and `MANUAL.md`; its commit list has nothing on networking):

- Library tabs **Internal**, **USB** (only when USB drives are on) and **Network**. L1/R1 go round the tabs, each tab keeps its position, the app reopens on the last tab.
- Shares are added by editing `network.cfg`; there is no screen for it.
- The Network tab scans once, keeps the list in a file, and contacts the share again only on **Square**, on a game start, or in Settings > Library > Scan for games. A sleeping NAS is not woken by browsing.
- An empty Network tab says what the share answered. A start waits for a slow NAS with "Waiting for the network share (12 s)" and Circle cancels.
- Settings > Library > "Load network games into memory" (on by default) reads a whole game before it starts.

## Decision

1. **Three tabs on the shelf** (`fe_shelf.cpp`): Internal (`!on_usb && !on_network`), USB (`on_usb`; shown only while a USB game exists), Network (`on_network`; always shown, so a new player finds it). The shelf keeps one game list and one cover service; a tab is a view (indices) onto it, so covers, scan and the CRC naming are unchanged.
2. **L1/R1 change tab**, round and round. Skipping ten games moves to **L2/R2**. Each tab keeps its selection while the shelf is up. The tab last used is saved as `last_tab` in `snes9x-ps5.ini` and opened first. Until the player changes tab, a tab left empty by a scan hands over to the first tab with games (a console with only network games opens on Internal and moves to Network once the share has connected). The Network tab is kept while a share is set up and still connecting, and it is where the shelf opens when no tab has games but a share is set up, because its message ("Could not reach the network share" and why) helps more than "No games found". The rules are `fe_tabs.h` (unit test `tests/test_tabs.cpp`); host test group 32 drives the real shelf.
3. **Count and sub line are per tab.** The `USB` / `NET` text after the region is dropped: the tab says it.
4. **The Network tab says what the shares are doing.** A line per share that is not Ready: `Connecting to <address>... 12 s`, or the failure sentence (`netfs_shares()`; never a password). With no games: no share set up (and where to add one: Triangle > Settings > Network shares), still connecting, failed, or connected with no ROMs found. This is PSFlycast's "the tab says what the share answered".
5. **Square on the Network tab scans again** (PSFlycast's Square): forgets the cached listings (`netfs_rescan`), reconnects the shares that are Failed or Dead (`netfs_reload`), then scans the shelf with the progress screen. On Internal and USB Square stays "Get cover". A network game's cover is then retried with R3 (Fix Covers).
6. **Not mirrored, and why**
   - *`network.cfg`*: shares stay in `shares.txt` with the Network shares screen ([ADR 003](./003-network-shares-screen.md)); that already goes beyond PSFlycast.
   - *A saved game list and no connection until the tab is opened*: shares connect in the background at start and a ROM is only listed once its share is Ready. Browsing the Network tab costs nothing extra; a share that is off is skipped within the 15 s connect limit. Revisit if Jason wants a sleeping NAS left alone (an option "Connect to shares: at start / when the Network tab opens").
   - *"Waiting for the network share" with Circle to cancel on a start*: there is no per-share cancel in netfs, and a ROM is never offered while its share is down. The 15 s limit applies.
   - *"Load network games into memory"*: a ROM is already fetched whole at launch ([ADR 001](./001-network-rom-loading.md)); there is no streaming mode to turn off.
   - *Search (R3), jump by letter (L2/R2), game details*: PSFlycast's library is a grid with those; the shelf is a cover flow and keeps its own keys (R3 stays Fix Covers).

## Alternatives considered

| Option | Why not |
|---|---|
| Two sections in one list (local, then network) | Needs a divider the cover flow does not have; one list of hundreds of covers still has to be scrolled past. |
| A separate "Network" screen reached from the shelf | A second place to find games; PSFlycast and the ask both want games in the library view. |
| Always show all three tabs | An empty USB tab on every console without a stick is noise; Network stays visible because it explains how to start. |

## Consequences

### Positive
- Local games are no longer mixed with games on a share, and the Network tab explains an empty or failing share without opening Settings.
- No change to netfs, the cover service or the scan.

### Negative
- L1/R1 no longer skip ten (L2/R2 now). Square on the Network tab no longer fetches the selected cover.
- A tab bar uses the top of the shelf; the version and author lines stay on the left.

### Neutral
- One more setting key (`last_tab`).
