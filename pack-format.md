# The `.achso` dictionary pack (format 1)

One file holds everything the app knows about German: over 50,000 lemmas with genders,
plurals, case forms, conjugation tables, verb objects, glosses and example sentences,
plus the curated drill tables. It is built by `pipeline/build_pack.py` from the
1.1 GB Wiktionary export and ships as **~24 MB** (≈12 MB LZFSE for the CloudKit
download). The app memory-maps it and reads it in place: nothing is unpacked,
nothing is decoded until a screen asks for it.

Ideas carried over from MarAI Keyboard's `.sym.bin`:

| idea | here |
|---|---|
| memory-map, never inflate into heap structures | `PackFile` maps the file, `POSIX_MADV_RANDOM` |
| FNV-1a 64 → XOR-folded 32-bit keys | surface index and delete index keys |
| ids are frequency ranks | lemma id 0 is the most common word; ties break toward common words |
| frequency-tiered SymSpell delete index (rule 86) | depth 2 for the 8,000 most common lemmas, depth 1 for the rest |
| `minDeleteLength` 1 (rule 86) | two-letter words stay reachable |
| store what is not recoverable (rule 73) | regular adjective endings are not stored; the app strips them |
| a format version is checked, a pack that fails it is refused | magic, format, CRC32 over the payload |

## Layout (little-endian)

```
header (24 bytes)
  0  "ACHS"
  4  u16 format (1)          6  u16 flags (0)
  8  u32 dataVersion (YYYYMMDD of the build; CloudKit updates compare this)
 12  u32 sectionCount       16  u32 CRC32 of everything after the header
 20  u32 reserved
section table: sectionCount × { tag[4], u32 offset, u32 length, u32 reserved }
sections, each 4-byte aligned
```

| tag | content |
|---|---|
| `STRS` | string pool: u16 length + UTF-8; a string ref is the byte offset of its length (0 = empty) |
| `LEMM` | 24-byte records: word, gloss, detailOffset, ipa (refs, u32), score u32, pos u8, gender u8 (m 1, f 2, n 4, plural-only 8), flags u8, level u8 |
| `DETL` | per lemma: (u16 tag, u16 count, u32 × count)… ending with tag 0. 1 senses, 2 noun forms (nom/gen/dat/acc sg, then pl), 3 verb forms (30 slots), 4 verb info, 5 prepositional objects, 6 case objects, 7 comparative/superlative, 8 examples (de, en, Tatoeba id, author, translation id, translation author), 9 compound head, 10 other plurals |
| `SURF` | u32 count, then (u32 key, u32 lemma<<4 | kind) sorted by key — every inflected spelling → its lemma |
| `DELS` | SymSpell index over folded headwords: header (K, P, prefixLength 7, maxQueryDistance 2, deepCore, deepDepth, tailDepth), keys u32[K], rows u32[K+1], postings u32[P] |
| `PREF` | lemma ids sorted by folded headword (prefix search) |
| `VPRP` | curated verb + preposition pairs with cloze sentences |
| `TWOW` | curated two-way preposition sentences |
| `PATT` | gender statistics per ending, counted over all Wiktionary nouns |
| `ADJA` | adjectives whose generated endings Wiktionary confirms (safe for drills) |
| `META` | JSON: version, counts, sources and licences, gloss counts per language |
| `GLtr`, `GLes`, `GLfr`, … | glosses in other languages from German Wiktionary: u32 lemma count, then one string ref per lemma id (0 = none, the app falls back to Apple Translation, then English). Keyed by the pack's own word ids, so an overlay is only valid with the pack it was built with |

Flags: 1 drillable, 2 weak noun (n-Deklination), 4 separable, 8 reflexive only,
16 homograph with another gender (der/die See), 32 irregular, 64 aux sein, 128 aux haben.

## Search

1. **Exact and inflected**: fold the query (lowercase, ß → ss, strip marks) and
   read it a second time with ae/oe/ue as umlauts; look both up in `SURF`.
   `ging` → gehen, `haeuser` → Haus, `strasse` → Straße.
2. **Adjective endings**: strip -en/-em/-er/-es/-e and look for an adjective base.
3. **Typos**: SymSpell over `DELS`, verified with optimal string alignment ≤ 2.

The Python reference (`reference_lookup`) writes `testvectors.json`; the Swift
tests must reproduce it exactly.

## Licence

The pack is a derived database of CC BY-SA 4.0 sources (Wiktionary, FrequencyWords)
and CC BY 2.0 FR sentences (Tatoeba; each German sentence and its translation credit
their authors). Share-alike most likely applies to the pack itself; the plan is to
publish the built pack openly alongside any App Store release. How CC BY-SA's
technical-restrictions clause interacts with App Store distribution is still an open
question to confirm before launch (this is not legal advice).
