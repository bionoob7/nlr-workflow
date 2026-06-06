# BibTeX Format Reference

Load this when parsing the .bib file produces unexpected results. Covers author
name parsing edge cases, field syntax variations, and special character handling.

---

## 1. Author Name Formats

BibTeX uses several conventions for the `author` field. Parse all of them correctly.

### Last, First format (most common in Zotero exports)
```
author = {Smith, John and Jones, Mary and Williams, Robert}
```
→ First author last name: "Smith"

### First Last format (less common)
```
author = {John Smith and Mary Jones}
```
→ First author last name: last token before " and" = "Smith"

### Mixed format
```
author = {Smith, John and Mary Jones}
```
→ First author uses Last, First format → last name = "Smith"

### Single author
```
author = {Smith, John}
author = {John Smith}
```
→ Parse accordingly

### Collaboration / consortium names
```
author = {{TCGA Research Network}}
author = {{The Cancer Genome Atlas Research Network}}
```
Double braces indicate a literal name (not Last, First). Store the full string.
For matching, try the first meaningful word: "TCGA" for {TCGA Research Network}.

### Names with particles (von, de, van, etc.)
```
author = {van der Berg, Jan}    → last name: "van der Berg" or "Berg" (either works)
author = {de la Cruz, Maria}    → last name: "de la Cruz" or "Cruz"
```
For matching: try stripping the particle first ("Berg" for van der Berg).
If no match, try with the particle ("vanderBerg").

### Non-ASCII characters
```
author = {M{\"u}ller, Hans}     → display: "Müller"
author = {Garc{\'{\i}}a, Juan}  → display: "García"
```
For matching, normalize by stripping the LaTeX encoding:
- `{\"u}` → "u" (or "ue" for German)
- `{\'e}` → "e"
The placeholder `[Muller2023]` should match `{M{\"u}ller, ...}`.
Simple approach: strip all `{`, `}`, `\`, and the character after `\` for matching.

---

## 2. Year Field Formats

### Standard
```
year = {2023}
year = {2023},
```
→ year = "2023"

### Date field (common in newer BibLaTeX entries)
```
date = {2023-05-14}
date = {2023},
```
→ Extract 4-digit year from the start: "2023"

### Year in press or preprint
```
year = {2023},
note = {In press}
```
→ year = "2023" (use year field as normal)

```
year = {2024},
eprint = {2401.12345}
```
→ year = "2024"

### Missing year
If no `year` or `date` field: try to extract a year from the `url` or `doi` fields.
If still not found, mark the entry as year = "unknown" — it will not match any placeholder.

---

## 3. Citekey Formats

Citekeys appear on the first line of each entry:

```
@article{smith2023towards,
@inproceedings{NEURIPS2023_abc123,
@misc{openai2023gpt4,
@book{hastie2001elements,
```

Extract everything between `{` and the first `,` on that line.
Citekeys are case-sensitive in BibTeX; use them exactly as written when constructing `[@citekey]`.

---

## 4. Title Field

Extract for disambiguation display only:

```
title = {Towards Multimodal Cancer Diagnosis},
title = {{GPT-4 Technical Report}},
title = {A Survey of Large Language Models for {NLP}},
```

Remove braces and take first 6 words. Strip LaTeX commands (e.g., `\textit{...}`).

---

## 5. Parsing Strategy for Large .bib Files

For .bib files with hundreds of entries, use Grep to pre-filter before full reading:

```bash
# Find all citekeys
grep -E "^@[a-z]+\{" library.bib

# Find all entries for a specific author
grep -iB1 "author.*Smith" library.bib

# Find all entries for a specific year  
grep -E "year\s*=\s*\{?2023" library.bib
```

Build the index incrementally: for each citekey line, read the next 15 lines to
extract author, year, and title. Stop when you hit the next `@` line.

---

## 6. Handling Malformed .bib Files

### Missing closing brace
```
@article{smith2023,
  author = {Smith, John,    ← missing closing brace
  year = {2023},
```
Parse as best you can; flag the entry in the report as "possibly malformed."

### Duplicate citekeys
```
@article{chen2023,  ...
@article{chen2023,  ...  ← duplicate
```
Both entries exist; store both, label as `chen2023` and `chen2023_dup`. Report to user.

### Entries with no author
```
@misc{anonymous2023,
  title = {Some report},
  year = {2023},
```
→ Set last name = "anonymous" or the first word of the title.

---

## 7. Multi-Cite Placeholder Parsing

When the manuscript has multiple citations in one bracket:

```
[Smith2023; Jones2024]
[Wang2023; Chen2024; Liu2025]
```

Split on `;` (trim whitespace around each part). Resolve each independently.
If any part is UNMATCHED, keep the whole bracket partially replaced:
```
[Smith2023; Jones2024]  →  [@smith2023towards; Jones2024]
```
The UNMATCHED placeholder stays as-is inside the bracket so it's visible for manual attention.

---

## 8. Placeholder Suffix → Candidate Index Mapping

| Placeholder suffix | Use candidate at index |
|---|---|
| (none) or `a` | 0 (first match in .bib order) |
| `_2` or `b` | 1 |
| `_3` or `c` | 2 |
| `_4` or `d` | 3 |

".bib file order" means the order the entries appear in the file from top to bottom.
This is usually chronological within the same author-year group.

If the suffix index is out of range (e.g., `_2` but only one entry exists), mark as
UNMATCHED rather than guessing.

---

## 9. Already-Converted Placeholders

Patterns to skip without processing:

- `[@smith2023]` — already in pandoc format
- `[@smith2023; @jones2024]` — already multi-cite pandoc format
- `^[1]^` or `[^1]` — footnote/endnote markers, not citations
- `[Figure 1]`, `[Table 2]` — cross-references, not citations

Detection: if the content inside `[...]` starts with `@`, it's already converted.
If it starts with `Figure`, `Table`, `Fig.`, `Eq.`, `Section`, skip it.
