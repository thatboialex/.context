# Plan: box art for renamed and fan-translated ROMs

Status: Implemented in a draft PR, 2026-10-17; host tests pass, console behaviour unverified. Requested by Jason. Code: `ps5/frontend/fe_artmatch.h`, `fe_romheader.h` and `fe_dbart.h` (new), `ps5/tools/make_extra_crc.py`, `make_db_art.py` and `data/snes-extra-crc.tsv`, `snes-db-art.tsv` (new), hooks in `fe_covers.cpp` and `fe_games.cpp`. Why it is shaped this way: [ADR 002](./decisions/002-box-art-name-matching.md).

## Goal

A fan-translated or renamed ROM finds its box art when the art exists under another name. It must work for any game, not one title. Example from Jason: `Super Robot Wars 4 (English).sfc` shows a placeholder, but the art source has `Dai-4-ji Super Robot Taisen (Japan)`.

Non-goals: new art sources, changing the title the shelf shows, guessing art for games the art source does not have.

## What the study found

1. **The name chain stops at "not found" for translations.** A ROM is named by exact No-Intro file name, then CRC32, then loose name, then edit-distance fuzzy (`fe_games.cpp`). A patched ROM has no known CRC, and "Super Robot Wars 4" is far in edit distance from "Dai-4-ji Super Robot Taisen".
2. **The art lookup repeats the same idea** (`MatchArtName`, `fe_covers.cpp`): a hand table of 8 titles (`fe_artaliases.h`), exact name, then the same fuzzy match. Each new translation needs a code edit.
3. **Fan translators change the name, not the cartridge.** Typical changes: `(English)`, `[T+Eng v1.1]`, `Final Fantasy 6` for `Final Fantasy VI`, `Wars` for `Taisen`, `Dai-4-ji` as `4`, words in another order, `Butoden` for `Butouden`, a dropped subtitle.
4. **The ROM header often survives.** The 21-byte internal title is usually left as the original Japanese release had it. It identifies the game when the file name does not.
5. **The art list is data we already ship**: `snes-cover-index.txt`, 3690 names, 1801 of them `(Japan)`.
6. **Online databases know patched ROMs by checksum.** Checked from the cloud session:
   - **Hasheous** (hasheous.org): anonymous, no key. Its SNES dump has 3812 games and, per ROM, name, size, CRC32, MD5 and SHA1. It lists fan translations: CRC `72BBACCA` is `Super Robot Wars 4 T-Eng (2026)(Aeon Genesis)`. 6364 of its CRCs are not in the No-Intro table.
   - Hasheous also maps a game to other databases and serves a cover for it: `GET /api/v1/images/<image id>` (an IGDB image, anonymous, JPEG) and, through the TheGamesDB id, `https://cdn.thegamesdb.net/images/original/boxart/front/<id>-1.jpg` (anonymous). Checked 2026-10-17: Aladdin 2000, Pokemon Stadium and King of Fighters 2000 have one; Street Fighter EX Plus Alpha has an id but its image is a 404.
   - **ScreenScraper** and **TheGamesDB's search API** need a developer key or account (the MetadataProxy name search answers 401). Not available here.
   - **libretro-database** (the source of our No-Intro table) has no translations.

## Design

Lookup order for a game's art name (first hit wins; every step refuses when ambiguous):

| # | Step | Status |
|---|---|---|
| 1 | Bootleg guard and hand aliases (`fe_artaliases.h`) | unchanged |
| 2 | Exact name (No-Intro name when known, else file name) | unchanged |
| 3 | **Extra CRC table**: the ROM's CRC32 names the dump (`Super Robot Wars 4 T-Eng ...`); step 5's word match turns that into an art name | new |
| 4 | Edit-distance fuzzy (`titles::Best`), now rejected when the numbers disagree | changed |
| 5 | **Normalised words** of the file name (`artmatch::Resolve`) | new |
| 6 | **ROM header title** (`artmatch::ResolveHeader`) for ROMs no table identified | new |
| 7 | **Database cover** (`fe_dbart.h`): only when steps 1 to 6 found no libretro art. The ROM's CRC32, or its name's words, finds a game in `snes-db-art.tsv`; the cover is downloaded from hasheous.org or TheGamesDB | new |

**Normalising (step 5).** Both the file name and each art name are reduced the same way:
- Drop extension, `# SNES` labels, all `(...)`, `[...]`, `{...}` tags, trailing `English`, `Translated`, `Patched`, `v1.1`, `T-Eng`. Keep region tags only as a preference.
- Roman numerals to digits (`VI` is `6`), `Dai-4-ji` to `4`, drop articles and `no`, `&` and `and` are equal.
- Collapse Japanese romanisation variants (`ou`, `oo`, `uu`; `shi`/`si`; `tsu`/`tu`; `fu`/`hu`) so `Butouden` equals `Butoden`.
- A short table of translation pairs is one word (`Taisen` and `Wars`, `Densetsu` and `Legend`, `Kidou` and `Mobile`, ...).

**Matching.** Same joined words wins at once. Otherwise score each candidate by the weight of matched words (rare words weigh more, counted over the art list; near-spellings count). A candidate must match every number, cover the query and the candidate, and beat the next different game by a clear margin. A second tier compares the main title (before ` - `) or the subtitle alone, at a lower score, for translations that drop or rename one half. Region preference: the tag if any, else USA, World, Europe, Japan; clean releases beat Beta, Proto, Sample, Rev, translation tags.

**Whole name first.** A translation's name often has the original's title plus a subtitle (`Front Mission - Gun Hazard`), and the art list also has the first game alone (`Front Mission`). The main-title tier scored `Front Mission (Japan)` as high as the right game, so the match was refused as ambiguous, and fuzzy once picked the wrong cover. `Resolve` now first asks which candidate explains the whole query best (the full name, or its main title when the candidate has no subtitle). One that fits at 0.85 or more and beats the next by 0.05 wins; otherwise the earlier scoring decides. Compared on the 6364 extra-table names, 38 results changed, all for the better (36 Gun Hazard dumps, `Super Star Wars - Return of the Jedi Redux`, `Spider-Man - Separation Anxiety`); a stricter rule that required the subtitle was tried and dropped because it lost legitimate matches (`Chrono Trigger - Retranslation`, `Street Fighter II - The World Warrior`).

**Number guard.** `Dragon Quest III` and `Dragon Quest I & II` have the same letters; the old fuzzy step matched them. Both sides having numbers and neither set containing the other now rejects the match (`NumbersConflict`), in `gamedb::Fuzzy` and in `MatchArtName`.

**Extra CRC table (step 3).** `ps5/frontend/data/snes-extra-crc.tsv`, built into the ELF like the No-Intro table. 6364 rows `CRC<TAB>dump name`, none in No-Intro, headerless CRCs only. A cached CRC is already computed at scan time, so this costs a hash-map lookup. Refresh: from `ps5/`, `python3 tools/make_extra_crc.py --date YYYY-MM-DD` (downloads the Hasheous dump; `--zip` takes a copy), then commit the file. The host test fails if a row overlaps No-Intro. The name is only read by the matcher, never shown.

**Header (step 6).** Read the internal header only for ROMs no table named, at scan time, and cache the title in `crc-cache.txt` (same size and mtime key). Find the header at `$7FC0`, `$FFC0` or `$40FFC0`, with or without a 512-byte copier header, by plausibility (title text, map mode, size byte, checksum pair; a bad checksum only costs points). The title (half-width katakana is romanised) must equal an art name's letters, or be the start of exactly one art title (headers are cut at 21 characters). Country byte `0` prefers `(Japan)`. Used only when the earlier steps found nothing, and never for the bootleg-guard titles.

**Database covers (step 7).** `ps5/frontend/data/snes-db-art.tsv`, 411 games, 32 KB, built into the ELF. A row is `name<TAB>sources<TAB>CRC,CRC`; a source is `h<image id>` (hasheous.org) or `t<id>` (TheGamesDB). `tools/make_db_art.py` writes it from the same Hasheous dump `make_extra_crc.py` reads and leaves out a game with no cover source, and a game libretro-thumbnails already has (one of its dumps is in No-Intro under a name in the cover list). The app finds a row by the ROM's CRC32 (`GameInfo::crc`, set at scan time), else by `artmatch::Resolve` over the row names, with the same refusals as step 5. `fe_covers.cpp` puts the database address after the libretro boxart address in the game's `wanted.txt` line, ahead of title and snap, and only when libretro has no boxart for the game. The helper already tries a line's addresses in order until one is not a 404, accepts PNG or JPEG by its first bytes and has no host list, so nothing in the helper or its protocol changed. Nothing about the ROM is sent: the table decides, the helper fetches an image by id. No live lookup: the whole SNES list is already in the table (see D4). Refresh: from `ps5/`, `python3 -I tools/make_db_art.py --date YYYY-MM-DD` (`--zip` takes a copy of the dump), then commit the file. Test overrides: `SNES9X_DBART_HASHEOUS` and `SNES9X_DBART_THEGAMESDB` replace the two image bases.

**What stays as is.** The shelf title, the CRC chain and the cover cache file names. The cover file is named after the matched art name, so a translation and its original share one download.

## Measured (host, 2026-10-17)

| Check | Result |
|---|---|
| ~44,400 renamed variants of every art name (tags, `(English)`, `[T+Eng]`, lower case, roman numerals, `Dai-N-ji`) | each resolves to its own game; none empty or wrong |
| ~55,000 variants in a wider sweep | 0 wrong, 0 empty |
| 2339 deliberately spoiled names | only ever their own game, or nothing |
| English translations in the extra table that find art | 262 of 333 (79%); the rest are games the art list does not have |
| Jason's 79 ROMs (from his 2.6 `boot.log`): missing before the database step | 7 by name: Aladdin 2000, Pokemon Stadium, King of Fighters 2000, Street Fighter EX Plus Alpha have a database cover now (Street Fighter EX Plus Alpha's TheGamesDB image was a 404 in the check, so it may stay blank); Street Fighter III - Super Version, Fire Emblem and Fire Emblem 3 still have none |
| `make -C ps5 test` | all passed (section 9c is new) |

## Acceptance

Automated, in `ps5/tests/test_titlematch.cpp` (already run by CI) and `make -C ps5 test` section 9b:

| # | Rule |
|---|---|
| 1 | `Super Robot Wars 4 (English)` and nine renamings of it resolve to `Dai-4-ji Super Robot Taisen (Japan)`. |
| 2 | Every name in the art list, with tags stripped, lower-cased and `(English)`/`[T+Eng]` added, resolves to its own game. |
| 3 | Near misses stay empty: `Super Robot Wars 5`, `Super Robot Wars` alone, a random title, `Dragon Quest III` against `I & II`. |
| 4 | Header reading finds LoROM, HiROM, ExHiROM and copier-header images; a bad checksum still works; garbage returns nothing. |
| 5 | CRC `72BBACCA` and `6B36672F` resolve to the Japanese Super Robot Wars 4 art; the table never overlaps No-Intro. |
| 6 | In the app flow: a forged CRC, a words-only name and a header-only ROM each get the Japanese cover. |
| 7 | Existing answers do not change: all earlier `titlematch` asserts still pass. |
| 8 | `Front Mission - Gun Hazard` (English, T-Eng) resolves to `Front Mission Series - Gun Hazard (Japan)`; `Front Mission T-Eng` to `Front Mission (Japan)`. |
| 9 | The database table: CRC `752A25D3` finds Aladdin 2000, `A8142249` Pokemon Stadium, `King of Fighters 2000` finds its row by words, `Street Fighter III - Super Version` finds nothing, and no CRC in it is a game libretro has art for. |
| 10 | In the app flow (`make test` 9c): a forged CRC downloads its cover from a fake hasheous.org (a PNG with no extension), a words-only name downloads a JPEG from a fake TheGamesDB, a game no database lists asks neither. |

Console behaviour (cover really appears on the shelf) is unverified until Jason confirms it on hardware.

## Open decisions (defaults chosen, change on request)

| # | Question | Default |
|---|---|---|
| D1 | A ROM hack of a known game (header says Super Mario World) shows that game's art when its name matches nothing. | Yes. A hack with that art is better than a blank card. |
| D2 | A header checksum index would identify patched ROMs whose translator left the checksum alone. | Not now: it needs the original ROM files. Revisit with a list Jason generates on his PC. |
| D3 | More hand aliases for English retitles that share no words with the Japanese name (`Treasure of the Rudras`). | A few verified ones, each checked against the art list by a test. |
| D4 | Ask Hasheous live for a CRC the embedded table lacks (a translation newer than the snapshot). | Not built. It needs a helper request type (the app has no HTTPS of its own), sends ROM checksums to a third party, and cannot be verified from the cloud. Jason chose "Not now" on 2026-10-17. Later that day he asked for online databases for the missing covers; that is step 7, built from the embedded dump instead of a live request, because the dump already lists every SNES game Hasheous knows. A live request would add only games newer than the snapshot. Refresh with the scripts. |
| D5 | Cover images come from hasheous.org and cdn.thegamesdb.net. | Unverified on the console: the cloud proxy re-signs TLS, so the certificates were not checked against the app's `cacert.pem`. If the helper log says "Certificate verification failed" for either host, the cover list for that host stays empty and the host test cannot show it. |
| D6 | Street Fighter III has no SNES release, so no database lists it. | No cover. Use a file next to the ROM, or `covers/<ROM file name>.png` or `.jpg`. |
