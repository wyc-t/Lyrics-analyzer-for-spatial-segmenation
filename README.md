# MXL × Lyrics Annotation Analyzer

A browser-based analysis tool for studying spatial segmentation of Cantonese MV lyric subtitles. It pairs MusicXML scores (`.mxl`) with hand-annotated spatial-break files (`.md`) and produces character-aligned visualisations, alignment checks, and per-song statistics.

Files stay on your machine. No backend, no upload, no telemetry.

---

## Requirements

A Chromium-based browser on macOS, Windows, or Linux:
- Chrome ≥ 86
- Edge ≥ 86
- Opera ≥ 72

The app uses the **File System Access API**, which Firefox and Safari do not implement. There is no fallback for those browsers — they cannot read a folder of files persistently.

---

## Project folder structure

A project is just a folder. Drop matching pairs of files into it:

```
my-project/
├── feng.mxl       ← MusicXML score
├── feng.md        ← spatial-break annotations
├── jyun.mxl
├── jyun.md
├── ...
└── project.json   ← created automatically; safe to hand-edit
```

The app pairs files by **filename stem** — `feng.mxl` with `feng.md`, `jyun.mxl` with `jyun.md`, etc.

`.musicxml` and `.xml` are also accepted in place of `.mxl` (same parser).

---

## MD annotation format

Each song's `.md` file contains one or more entries, one per MV version:

```markdown
**[1.1]** 記不記得/你將證件/與機票/連著一份渴望//放入這背囊//紮根也好

**[1.2]** Official MV: https://www.youtube.com/watch?v=...
記不記得/你將證件/與機票連著/一份渴望//放入這背囊紮根/也好

**[1.3]** Karaoke version
記不記得你將證件/與機票/連著一份渴望放入這背囊//紮根也好
```

### Rules

- **Entry header**: `**[N.X]**` where `N` is the song number and `X` is the version number. Example: `[1.1]`, `[1.11]`, `[2.6]`.
- **Lyric line**: may follow on the same line as the header, or on the next line. Anything else on the header line after `**[N.X]**` is treated as metadata (URL, comment, etc.) and shown in the Alignment view.
- **`/`** = spatial break. A segmentation boundary between lyric groups on screen, within a single subtitle line.
- **`//`** = line break. New display line. A `//` is also counted as a spatial break.
- **Whitespace**: ignored inside the lyric line. You can indent or wrap freely.
- **Characters**: any non-`/`, non-whitespace Unicode codepoint counts as one "character". One Chinese character = one syllable = one note (with melisma handled — see below).

### Ignored entries

Entries that use `.` notation between CJK characters (e.g. `記不記得.你將證件`) and contain no `/` are recognised as a **different annotation type** and are skipped. They appear in the Alignment view marked "ignored" so you can audit them, but they are not used for break statistics.

### Alignment

Stripping all `/` and `//` from an entry gives a plain character sequence. This must match the lyric sequence extracted from the MXL **character-by-character**. The Alignment tab flags any mismatch — length difference or character substitution at any position — in red, so you can fix the source file. Common cause: variant characters (`散` vs `撒`, `一` vs `壹`, etc.).

---

## MXL parsing notes

- The first part containing any `<lyric>` is treated as the vocal part. If no part has lyrics, the first part is used.
- Each note carrying a `<lyric><text>` is a **character entry** with that character.
- Notes with no lyric that follow a lyric-bearing note (with no rest in between) are treated as **melisma continuations**: their duration is folded into the preceding character. The Character view shows a melisma badge and the cell height reflects the summed duration.
- Notes with no lyric that follow a rest are treated as **instrumental** notes — shown as hatched cells, not counted as characters.
- Rests are shown as hatched cells but do not contribute to the character index used for alignment.
- Durations are converted from MusicXML divisions to eighth-note units (×2 of quarter-length).
- Multi-pitch chord members (`<chord/>` elements) and grace notes are skipped.
- Time signature: first `<attributes><time>` encountered.
- Tempo: first `<sound tempo="…">` or `<metronome><per-minute>` encountered.

---

## project.json

A manifest file written automatically into the project folder. Currently used for:

```json
{
  "version": 1,
  "songs": {
    "feng": {
      "title": "風",
      "composer": "...",
      "notes": "..."
    }
  },
  "pairings": {
    "song1.mxl": "lyrics_v1.md"
  }
}
```

- `songs` — keyed by filename stem. Add a `title` field to display a human-readable Chinese title in the sidebar and song header. Other fields are reserved for future UI use.
- `pairings` — optional map of `mxl_filename → md_filename` for **manual pairings** when stem-matching fails. The app reads this on every scan.

Hand-editing the file is supported. The app rewrites it when it has changes to record; otherwise it leaves your edits alone.

---

## Views

**Character.** One cell per character in lyric order; height proportional to note duration in eighth-note units. Local-duration-maxima cells are highlighted. Right-edge ticks show spatial breaks for each enabled MV version (one tick per version; line breaks shown in a different colour). The heatmap row beneath shows break frequency per position. Click any cell for full per-character detail.

**Measure.** Same data, organised one row per measure. Useful for seeing how breaks distribute against musical phrase structure. Click any cell to jump to the same character in the Character view.

**Alignment.** For each MD entry: side-by-side display of the MXL character sequence and the MD character sequence, with mismatches highlighted. The header badge counts the number of entries that fail alignment.

**Stats.** Per-song summary: total characters, version count, local-maxima count, positions with ≥1 break, unanimous-break positions, local-maxima with no break, breaks at non-local-maxima, and a duration distribution histogram. The two crosstab counts ("local maxima with no break", "breaks at non-local-maxima") correspond to the key empirical claims of the paper.

---

## Version filter

In Character and Measure views, the version-filter row at the top toggles which MV versions contribute to the break visualisation. Click `all` to toggle everything. Ignored entries (dot-notation) are shown greyed out and cannot be enabled. Stats are computed over **all non-ignored versions** regardless of the filter — the filter only affects the visualisation, not the numbers.

---

## Limitations

- The app reads only the first lyric verse (`<lyric number="1">`). Multi-verse scores will lose alternative verses.
- Melisma detection uses a heuristic: a note with no `<lyric>` that follows a lyric-bearing note with no intervening rest is treated as a continuation. If your MusicXML uses unusual conventions (e.g. empty `<lyric>` elements on continuation notes), the count may be off — check the Alignment view.
- Duration is taken from `<duration>` divided by `<divisions>`. Tuplets and grace-note rebasing are handled implicitly by MusicXML's divisions; ties are not collapsed (a tied half + quarter shows as two adjacent cells with the second melisma'd).
- Browser permissions for the project folder are session-scoped. You will be prompted to re-grant permission when reopening the app.

---

## Troubleshooting

**"No MusicXML file found inside MXL"** — the `.mxl` file is corrupt or missing its `META-INF/container.xml`. Re-export from MuseScore / Finale / Sibelius.

**Alignment fails at position N** — open the Alignment view, locate the red character, and fix it in either the MXL or the MD file. Then click **Rescan**. Common culprits: variant Chinese characters, missing/extra characters in the MD that don't match the score.

**Character count off after melisma** — check whether the MXL has an empty `<lyric>` element on continuation notes (some exports do this). The current heuristic treats a note with no `<lyric>` as a melisma continuation; an empty `<lyric><text></text></lyric>` is also treated as no lyric (we trim and check non-empty).

---

## Hosting on GitHub Pages

Drop `index.html` into the root of any GitHub Pages-enabled repo. No build step. The File System Access API works over `https://` and over `http://localhost`, so the GitHub Pages URL is sufficient.
