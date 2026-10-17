# Plan: box art for renamed and fan-translated ROMs

Status: Proposed, 2026-10-17. Requested by Jason. Code: `ps5/frontend/fe_artmatch.h`, `fe_romheader.h` (new), hooks in `fe_covers.cpp` and `fe_games.cpp`. Why it is shaped this way: [ADR 002](./decisions/002-box-art-name-matching.md).

## Goal

A fan-translated or renamed ROM finds its box art when the art exists under another name. Example from Jason: `Super Robot Wars 4 (English).sfc` shows a placeholder, but the art source has `Dai-4-ji Super Robot Taisen (Japan)`.

Non-goals: new art sources, scraping websites, changing the title the shelf shows, guessing art for games the art source does not have.

## What the study found

1. **The name chain stops at "not found" for translations.** Today a ROM is named by exact No-Intro file name, then CRC32, then loose name, then edit-distance fuzzy (`fe_games.cpp:419-441`). A patched ROM has no known CRC, and "Super Robot Wars 4" is far in edit distance from "Dai-4-ji Super Robot Taisen".
2. **The art lookup repeats the same idea** (`MatchArtName`, `fe_covers.cpp:91`): a hand table of 8 titles (`fe_artaliases.h`), exact name, then the same fuzzy match. Each new translation needs a code edit.
3. **Fan translators change the name, not the cartridge.** Typical changes: `(English)`, `[T+Eng v1.1]`, `Final Fantasy 6` for `Final Fantasy VI`, `Wars` for `Taisen`, `Dai-4-ji` as `4`, words in another order, `Butoden` for `Butouden`, a dropped subtitle.
4. **The ROM header often survives.** The 21-byte internal title is usually left as the original Japanese release had it. It identifies the game when the file name does not.
5. **The art list is data we already ship**: `snes-cover-index.txt`, 3690 names, 1801 of them `(Japan)`. It even holds a few `[T-En by ...]` entries.
6. **No CRC route exists for patched ROMs.** The No-Intro table has no translations, and the original's CRC cannot be recovered from a patched file.

## Design

Lookup order for a game's art name (first hit wins; every step is skipped if it would be ambiguous):

| # | Step | Status |
|---|---|---|
| 1 | Bootleg guard and hand aliases (`fe_artaliases.h`) | unchanged |
| 2 | Exact name (No-Intro name when known, else file name) | unchanged |
| 3 | Edit-distance fuzzy (`titles::Best`) | unchanged |
| 4 | **Normalised words** (`artmatch::Resolve`): same words in the same order, then a weighted word-set match | new |
| 5 | **ROM header title** (`artmatch::ResolveHeader`) for ROMs the name chain could not identify | new |

**Normalising (step 4).** Both the file name and each art name are reduced the same way:
- Drop extension, `# SNES` labels, all `(...)`, `[...]`, `{...}` tags, trailing `English`, `Translated`, `Patched`, `v1.1`. Keep region tags only as a preference.
- Roman numerals to digits (`VI` is `6`), `Dai-4-ji` to `4`, drop articles and `no`, `&` and `and` are equal.
- Collapse Japanese romanisation variants (`ou`, `oo`, `uu`; `shi`/`si`; `tsu`/`tu`; `fu`/`hu`) so `Butouden` equals `Butoden`.
- A short table of translation pairs is one word (`Taisen` and `Wars`, `Densetsu` and `Legend`, `Monogatari` and `Story`, ...).

**Matching.** Same joined words wins at once. Otherwise score each candidate by the weight of matched words, where rare words (found by counting over the art list) weigh more. A candidate must match every number, cover the query and the candidate (or its main title before ` - `), and beat the next different game by a clear margin. Region preference: the tag if any, else USA, World, Europe, Japan; clean releases beat Beta, Proto, Sample, Rev.

**Header (step 5).** Read the internal header only for ROMs left unnamed, at scan time, and cache the title in `crc-cache.txt` (same size and mtime key). Find the header at `$7FC0`, `$FFC0` or `$40FFC0`, with or without a 512-byte copier header, by plausibility (title text, map mode, size byte, checksum pair). The title (half-width katakana is romanised) must equal an art name's letters, or be the start of exactly one art title (headers are cut at 21 characters). Country byte `0` prefers `(Japan)`. Header art is used only when step 4 found nothing, and never for the bootleg-guard titles.

**What stays as is.** The shelf title, the CRC chain and the cover cache file names. The cover file is named after the matched art name, so a translation and its original share one download.

## Acceptance

Automated, in `ps5/tests/test_titlematch.cpp` (already run by CI) and `make -C ps5 test`:

| # | Rule |
|---|---|
| 1 | `Super Robot Wars 4 (English)` and nine renamings of it resolve to `Dai-4-ji Super Robot Taisen (Japan)`. |
| 2 | Every name in the art list, with tags stripped, lower-cased and `(English)`/`[T+Eng]` added, resolves to its own game. |
| 3 | Near misses stay empty: `Super Robot Wars 5`, `Super Robot Wars` alone, a random title. |
| 4 | Header reading finds LoROM, HiROM, ExHiROM and copier-header images; a header with a bad checksum still works; garbage returns nothing. |
| 5 | Existing answers do not change: all current `titlematch` asserts still pass. |

Console behaviour (cover really appears on the shelf) is unverified until Jason confirms it on hardware.

## Open decisions (defaults chosen, change on request)

| # | Question | Default |
|---|---|---|
| D1 | A ROM hack of a known game (header says Super Mario World) shows that game's art when its name matches nothing. | Yes. A hack with that art is better than a blank card. |
| D2 | A header checksum index would identify patched ROMs whose translator left the checksum alone. | Not now: it needs the original ROM files, which a cloud session does not have. Revisit with a list Jason generates on his PC. |
| D3 | Add more hand aliases for English retitles that share no words with the Japanese name (`Treasure of the Rudras`). | A few verified ones, each checked against the art list by a test. |
