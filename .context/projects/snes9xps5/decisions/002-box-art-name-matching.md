# ADR-002: Find box art for translated ROMs by normalised words, then the ROM header

## Status
Proposed, 2026-10-17. Becomes Accepted when Jason has run it on the console with his translated library.

## Context

Jason's fan-translated ROMs (`Super Robot Wars 4 (English)`) get no box art, though the art source has the game under its Japanese No-Intro name. The current lookup forgives spelling only ([plan](../box-art-lookup-plan.md), "What the study found"). The art list is the embedded Libretro filename index; ROM bytes of a patched game match no CRC.

## Decision

1. Add a word-based matcher, `ps5/frontend/fe_artmatch.h`, after the existing exact and fuzzy steps. It reduces file name and art names to the same words (tags, roman numerals, `Dai-N-ji`, articles, romanisation variants, a short list of translation pairs), then matches in two tiers: equal words in order, then a weighted word-set match with a margin.
2. Refuse when unsure: every number must match, the best game must beat the next one clearly, and one ambiguous result returns nothing. A missing cover is acceptable; another game's cover is not.
3. For ROMs whose name no table could identify, read the internal header title (`fe_romheader.h`) at scan time, cache it beside the CRC cache, and use it as the last step of the art lookup only. It never changes the shelf title.
4. Keep the existing hand alias table and bootleg guard first. They stay the place for English retitles that share no words with the original.
5. Keep the matcher header-only and free of console code, so the standalone test that CI already runs covers it.

## Alternatives considered

| Option | Why not |
|---|---|
| Widen the edit distance in `titles::Best` | `Super Robot Wars 4` is 14 edits from `Dai-4-ji Super Robot Taisen`; a limit that large matches wrong games. |
| Grow the hand alias table | Every new translation needs a code change and a release. Kept only for the leftovers. |
| Hash the header checksum | Needs a table built from original ROMs. A cloud session has none. Possible later from a list Jason makes (plan D2). |
| Look the name up on a website | Adds a network dependency and a new data source for a problem the shipped list already solves. |
| Rename the game to the matched name on the shelf | Jason named the file; the shelf should show his name. |

## Consequences

### Positive
- Most translations resolve with no edit when their words are reasonable.
- Wrong matches are rare by construction (numbers, margin, ambiguity refusal).
- Header art helps ROMs whose file name says nothing.

### Negative
- Translation pairs and romanisation rules are a judgement list; some titles will still miss.
- First scan of a library with many unnamed ROMs reads each ROM's header once (cached after).
- A ROM hack shows the base game's art (plan D1).

### Neutral
- Cover files keep their art-name file names, so a translation and its original share one download.
