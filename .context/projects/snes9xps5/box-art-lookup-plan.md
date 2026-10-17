# Plan: box art for renamed and fan-translated ROMs

Status: Implemented in a draft PR, 2026-10-17; host tests pass, console behaviour unverified. Requested by Jason. Code: `ps5/frontend/fe_artmatch.h`, `fe_romheader.h` (new), `ps5/tools/make_extra_crc.py` and `data/snes-extra-crc.tsv` (new), hooks in `fe_covers.cpp` and `fe_games.cpp`. Why it is shaped this way: [ADR 002](./decisions/002-box-art-name-matching.md).

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
   - **ScreenScraper** and **TheGamesDB** need a developer key or account. Not available here.
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

**Normalising (step 5).** Both the file name and each art name are reduced the same way:
- Drop extension, `# SNES` labels, all `(...)`, `[...]`, `{...}` tags, trailing `English`, `Translated`, `Patched`, `v1.1`, `T-Eng`. Keep region tags only as a preference.
- Roman numerals to digits (`VI` is `6`), `Dai-4-ji` to `4`, drop articles and `no`, `&` and `and` are equal.
- Collapse Japanese romanisation variants (`ou`, `oo`, `uu`; `shi`/`si`; `tsu`/`tu`; `fu`/`hu`) so `Butouden` equals `Butoden`.
- A short table of translation pairs is one word (`Taisen` and `Wars`, `Densetsu` and `Legend`, `Kidou` and `Mobile`, ...).

**Matching.** Same joined words wins at once. Otherwise score each candidate by the weight of matched words (rare words weigh more, counted over the art list; near-spellings count). A candidate must match every number, cover the query and the candidate, and beat the next different game by a clear margin. A second tier compares the main title (before ` - `) or the subtitle alone, at a lower score, for translations that drop or rename one half. Region preference: the tag if any, else USA, World, Europe, Japan; clean releases beat Beta, Proto, Sample, Rev, translation tags.

**Number guard.** `Dragon Quest III` and `Dragon Quest I & II` have the same letters; the old fuzzy step matched them. Both sides having numbers and neither set containing the other now rejects the match (`NumbersConflict`), in `gamedb::Fuzzy` and in `MatchArtName`.

**Extra CRC table (step 3).** `ps5/frontend/data/snes-extra-crc.tsv`, built into the ELF like the No-Intro table. 6364 rows `CRC<TAB>dump name`, none in No-Intro, headerless CRCs only. A cached CRC is already computed at scan time, so this costs a hash-map lookup. Refresh: from `ps5/`, `python3 tools/make_extra_crc.py --date YYYY-MM-DD` (downloads the Hasheous dump; `--zip` takes a copy), then commit the file. The host test fails if a row overlaps No-Intro. The name is only read by the matcher, never shown.

**Header (step 6).** Read the internal header only for ROMs no table named, at scan time, and cache the title in `crc-cache.txt` (same size and mtime key). Find the header at `$7FC0`, `$FFC0` or `$40FFC0`, with or without a 512-byte copier header, by plausibility (title text, map mode, size byte, checksum pair; a bad checksum only costs points). The title (half-width katakana is romanised) must equal an art name's letters, or be the start of exactly one art title (headers are cut at 21 characters). Country byte `0` prefers `(Japan)`. Used only when the earlier steps found nothing, and never for the bootleg-guard titles.

**What stays as is.** The shelf title, the CRC chain and the cover cache file names. The cover file is named after the matched art name, so a translation and its original share one download.

## Measured (host, 2026-10-17)

| Check | Result |
|---|---|
| ~44,400 renamed variants of every art name (tags, `(English)`, `[T+Eng]`, lower case, roman numerals, `Dai-N-ji`) | each resolves to its own game; none empty or wrong |
| ~55,000 variants in a wider sweep | 0 wrong, 0 empty |
| 2339 deliberately spoiled names | only ever their own game, or nothing |
| English translations in the extra table that find art | 262 of 333 (79%); the rest are games the art list does not have |
| `make -C ps5 test` | 227 passed, 0 failed |

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

Console behaviour (cover really appears on the shelf) is unverified until Jason confirms it on hardware.

## Open decisions (defaults chosen, change on request)

| # | Question | Default |
|---|---|---|
| D1 | A ROM hack of a known game (header says Super Mario World) shows that game's art when its name matches nothing. | Yes. A hack with that art is better than a blank card. |
| D2 | A header checksum index would identify patched ROMs whose translator left the checksum alone. | Not now: it needs the original ROM files. Revisit with a list Jason generates on his PC. |
| D3 | More hand aliases for English retitles that share no words with the Japanese name (`Treasure of the Rudras`). | A few verified ones, each checked against the art list by a test. |
| D4 | Ask Hasheous live for a CRC the embedded table lacks (a translation newer than the snapshot). | Not built. It needs a helper request type (the app has no HTTPS of its own), sends ROM checksums to a third party, and cannot be verified from the cloud. Jason chose "Not now" on 2026-10-17. Build it opt-in only if he asks; until then refresh the table with the script. |
