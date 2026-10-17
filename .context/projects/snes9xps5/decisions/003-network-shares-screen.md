# ADR-003: Add and edit network shares on screen, with a pad-driven keyboard

## Status
Proposed, 2026-10-17. Becomes Accepted when Jason has used the screen on the console. The screen is unverified on hardware.

## Context

[ADR 001](./001-network-rom-loading.md) put shares in `/data/snes9x/shares.txt` and left an on-screen keyboard to Phase 5. Jason asked for the visual screen now: type, server, share or export, user name and password, with a connection test, edit and remove, so nobody has to edit a text file over FTP.

- The app has no text input at all ([architecture.md](../architecture.md), "Build and test").
- A native title links only the system modules listed in `ps5/proto/native/tooling/native/runtime/imports.txt`. That list is byte-identical to SwanStationPS5's and has no `libSceImeDialog`.
- The payload SDK is not in the cloud container, so the dialog's header, its module load and its overlay behaviour in a title that draws its own frames cannot be checked, and the console is not reachable from a cloud session.
- The netfs core already parses `shares.txt` and mounts from it (PR thatboialex/snes9xPS5#7).

## Decision

1. **Settings > Network shares** (`fe_netscreen.cpp`) lists each share with its state and has a form: type (SMB or NFS), server, share name or export path, folder, user name, password, and for SMB the domain, version, signing and encryption (for NFS the version, user ID and group ID). Test connection, Save, Remove and Cancel are rows of the form. `nfsport` and `mountport` have no field; they are kept when a share is edited.
2. **Keyboard.** `fe_osk.cpp` draws a keyboard and the pad drives it (D-pad and Cross; Square deletes, Triangle types a space, L1/R1 move the cursor, L2 is Shift for one key, R2 the symbols page, OPTIONS is Done, Circle cancels). The system IME dialog is not used because it cannot be linked or tried from here. Replace the body of `EditText` with it later if it proves to work; nothing else calls the keyboard.
3. **`shares.txt` stays the single store.** The screen changes one line (replace, append or delete) and leaves every other byte alone: comments, order, line endings, a BOM, and a `# note` after an address. Writes go through `<file>.part` and a rename. A line is found again by its line number and mount point just before the write; if the file changed meanwhile (FTP), the screen refuses and says so. This replaces the plan's rule that the app never rewrites a file it did not create: the app still never rewrites at load, only when the player saves or removes.
4. **Saving reloads netfs** (`netfs_reload`): every share disconnects and connects again, and the shelf picks the games up as it does after boot.
5. **Test connection** (`NetfsConnectionTest`) connects on its own thread with its own backend, lists the configured folder, reports "Connected. The folder has N files and M folders." or the failure sentence, and disconnects. The screen polls it each frame, the time limit is the connect limit (15 s) and Circle stops it. It never touches the shares that are mounted.
6. **Passwords.** Drawn as `********` on the form and as `*` on the keyboard, with no reveal. The password is written to `shares.txt` percent-encoded as `?password=`, nowhere else: it reaches no log, notice or message (a host test greps the logs for it). The README already says the file is plain text on `/data`.
7. **Validation reuses the address parser.** The screen builds the address text from the fields and parses it with `netfs_parse_share`, so the file, the screen and the loader agree on what is valid. Fields that go into the address unescaped (the server) are checked first so they cannot smuggle a `/`, `@` or `?`.

## Alternatives considered

| Option | Why not |
|---|---|
| System IME dialog (`libSceImeDialog`) | Not in the imports a native title links; header, module load and overlay unverifiable without the SDK and a console. Revisit once a console session can try it. |
| Keep `shares.txt` read-only and add the screen later | Jason asked for the screen; a file edited over FTP is the thing he wants gone. |
| Store shares in `snes9x-ps5.ini` | Two stores to keep in sync, and hand-written lines and notes would be lost. |
| Rewrite the whole file from the parsed shares | Loses comments, notes, and lines that did not parse. |
| Reveal the password while typing | Contradicts "masked on screen". The keyboard shows each key as it is pressed. |

## Consequences

### Positive
- A share is added with the pad alone. A wrong server, share or password shows its own sentence before the share is saved.
- Hand-edited files keep working, and edits made on the screen keep a hand-written file readable.
- The screen needs no backend change: it uses the same `netfs::Backend` as the mount.

### Negative
- Typing with a pad is slow. Long passwords are tedious; hosts and users on one keyboard page are quick.
- Saving reconnects every share, not only the one that changed.
- A hand-written line past the eighth, or one that repeats a folder, is listed as "Not used" and has to be removed by hand or on the screen.
- Unverified on the console: layout at 1080p and the pad feel have only been seen in host renders.

### Neutral
- `fe_ui.cpp` now holds the colours, pad reader, header, footer and option box that `fe_menu.cpp` kept private.
- Settings has one more row, so the walking-distance tests (groups 18 and 25) move by one press.
