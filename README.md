# imozuru-db

The Japanese vocabulary, kanji and example-sentence database used by the Imozuru iOS
app, published as a single read-only SQLite file.

**Download:** see [Releases](https://github.com/penteado-cloud/imozuru-db/releases).
Each release lists the source versions it was built from and the file's SHA-256.

## Licence

The database is an adapted work, licensed under
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) (full text in
[LICENSE](LICENSE)). If you share it or a work based on it, you must credit the
sources below and use the same licence. Third-party notices are in [NOTICE.md](NOTICE.md).

The same licence and attribution text is stored in the file itself:

```sh
sqlite3 imozuru.sqlite "SELECT key, value FROM meta"
```

## Sources

| Source | Creator | Licence | Used for |
|---|---|---|---|
| [JMdict](https://www.edrdg.org/jmdict/j_jmdict.html) | Electronic Dictionary Research and Development Group | [CC BY-SA 4.0](https://www.edrdg.org/edrdg/licence.html) | words, readings, senses, glosses |
| [KANJIDIC2](https://www.edrdg.org/wiki/index.php/KANJIDIC_Project) | Electronic Dictionary Research and Development Group | [CC BY-SA 4.0](https://www.edrdg.org/edrdg/licence.html) | kanji meanings, readings, grade, strokes |
| [jiten.moe frequency lists](https://jiten.moe/frequency-dictionaries) | jiten.moe | CC BY-SA 4.0 (Terms §6.1) | word ranking |
| [Tatoeba](https://tatoeba.org) | Tatoeba contributors | [CC BY 2.0 FR](https://creativecommons.org/licenses/by/2.0/fr/) | example sentences and translations |
| [JmdictFurigana](https://github.com/Doublevil/JmdictFurigana) | Doublevil | MIT (see NOTICE.md) | word furigana |
| [SudachiDict](https://github.com/WorksApplications/SudachiDict) | Works Applications | Apache-2.0 | sentence segmentation and readings |

`sentence.id` is the Tatoeba sentence ID. Its contributors are listed at
`https://tatoeba.org/sentences/show/<id>`.

## Changes from the sources

- Words are selected and ordered by jiten.moe frequency, matched to JMdict entries by
  (form, reading). Only the senses relevant to the chosen reading are kept, and phrases
  are flagged and re-ranked.
- Kanji are limited to those used by the selected words.
- Japanese Tatoeba sentences are paired with an English translation, filtered by rating,
  tags and owner proficiency, linked to words (from the Tatoeba indices and from Sudachi
  tokenization), and capped per word.
- Furigana for words comes from JmdictFurigana. Furigana for sentences comes from
  tokenizer readings.

## Schema

All JSON columns are compact UTF-8 JSON. `word.id` isn't stable across builds; use
`word.key` (`ent_seq|form|reading`). `meta` holds `schema_version`, `data_version`,
licence fields and the source checksums.

```sql
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL) WITHOUT ROWID;

CREATE TABLE kanji (
    id INTEGER PRIMARY KEY,
    literal TEXT NOT NULL UNIQUE,
    meanings TEXT NOT NULL,      -- JSON array
    on_readings TEXT NOT NULL,   -- JSON array, katakana
    kun_readings TEXT NOT NULL,  -- JSON array, "." marks okurigana
    grade INTEGER,
    strokes INTEGER NOT NULL,
    kd_freq INTEGER
);

CREATE TABLE word (
    id INTEGER PRIMARY KEY,
    key TEXT NOT NULL UNIQUE,    -- "ent_seq|form|reading", stable across builds
    ent_seq INTEGER NOT NULL,    -- JMdict entry number
    form TEXT NOT NULL,
    reading TEXT NOT NULL,
    rank INTEGER NOT NULL,       -- jiten rank (shared in the long tail)
    ordinal INTEGER NOT NULL,    -- position in the teaching order
    homograph_rank INTEGER NOT NULL,
    is_uk INTEGER NOT NULL,      -- usually written in kana
    furigana TEXT NOT NULL,      -- JSON [[start, length, ruby], ...]
    is_expression INTEGER NOT NULL
);

CREATE TABLE word_kanji (
    word_id INTEGER NOT NULL REFERENCES word(id),
    kanji_id INTEGER NOT NULL REFERENCES kanji(id),
    position INTEGER NOT NULL,
    PRIMARY KEY (word_id, kanji_id)
) WITHOUT ROWID;

CREATE TABLE sense (
    word_id INTEGER NOT NULL REFERENCES word(id),
    ord INTEGER NOT NULL,        -- JMdict sense number (1-based) within the entry
    pos TEXT NOT NULL,           -- JSON array of JMdict codes
    glosses TEXT NOT NULL,       -- JSON array
    misc TEXT NOT NULL,          -- JSON array
    info TEXT NOT NULL,          -- JSON array
    PRIMARY KEY (word_id, ord)
) WITHOUT ROWID;

CREATE TABLE sentence (
    id INTEGER PRIMARY KEY,      -- Tatoeba sentence id
    ja TEXT NOT NULL,
    en TEXT NOT NULL,
    kanji TEXT NOT NULL,         -- distinct kanji in ja
    furigana TEXT NOT NULL       -- JSON [[start, length, ruby], ...]
);

CREATE TABLE sentence_word (
    sentence_id INTEGER NOT NULL REFERENCES sentence(id),
    word_id INTEGER NOT NULL REFERENCES word(id),
    start INTEGER NOT NULL,      -- offset into ja in Unicode scalars
    length INTEGER NOT NULL,
    sense INTEGER,               -- JMdict sense number, when known
    checked INTEGER NOT NULL,    -- marked as a good example in the Tatoeba indices
    PRIMARY KEY (word_id, sentence_id)
) WITHOUT ROWID;
```

The tools that build this file are not part of this repository.
