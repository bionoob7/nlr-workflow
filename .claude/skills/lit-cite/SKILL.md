---
name: lit-cite
description: >
  Converts author-year citation placeholders in a manuscript draft to pandoc citekey
  format, matched against a BibTeX (.bib) file. Handles the full replacement pipeline:
  parse the .bib file to build an author+year → citekey index, scan every placeholder
  in the manuscript, auto-replace unambiguous matches, collect ambiguous cases and ask
  the user to resolve them as a batch, then save the cited manuscript. Use this skill
  whenever a manuscript contains [Author20XX] or [Author20XX_N] placeholders that need
  to be converted to [@citekey] for pandoc or Zotero workflow. Works for any
  markdown manuscript + BibTeX library combination. Invoke with /lit-cite.
allowed-tools: Read Glob Grep Write Bash
metadata:
  version: "1.0"
---

# Lit-Cite — Citation Placeholder Replacer

This skill converts author-year citation placeholders to pandoc citekey format in
two passes: a safe read-only pass that builds a replacement plan, followed by a write
pass that executes it. Ambiguous cases are batched and shown to the user before any
file is modified.

The key challenge is that `[Smith2023]` in the manuscript and `smith2023towards` in
the .bib file refer to the same paper, but no simple string match connects them. This
skill bridges that gap by extracting the first-author last name and year from both
sides and matching on those, then using titles and journal names to break ties.

Read `references/bib-formats.md` if you encounter an unusual .bib format or need
guidance on how to parse specific BibTeX field syntax.

---

## Step 1 — Locate the Manuscript and .bib File

**Manuscript:** Check in this order:
1. Path the user explicitly named
2. `manuscript/manuscript_v2.md` (post-polish version, preferred)
3. `manuscript/manuscript_cited.md`
4. `manuscript/manuscript_v1.md`
5. Any `manuscript/*.md` file (ask if multiple)

**BibTeX library:** Check in this order:
1. Path the user explicitly named
2. `references/library.bib`
3. `references/*.bib` (any .bib file in references/)
4. Any `*.bib` file in the project root or its immediate subdirectories
5. If not found: tell the user and stop — the skill cannot proceed without a .bib file

**Output path:** Save the result as `manuscript_cited.md` in the same directory as
the input manuscript. If the input is already named `manuscript_cited.md`, save as
`manuscript_cited_v2.md`. Never overwrite the source file.

State the three paths before proceeding: "Manuscript: ... | BibTeX: ... | Output: ..."

---

## Step 2 — Build the BibTeX Index

Read the entire .bib file. For each entry, extract:

1. **Citekey** — the identifier after `@type{`, before the first comma:
   ```
   @article{smith2023towards,   → citekey = "smith2023towards"
   ```

2. **First author last name** — from the `author = {...}` field:
   - Format `Last, First and ...` → first author last name = first token before comma
   - Format `First Last and ...` → first author last name = last token before " and"
   - Clean: remove accents for matching purposes (Müller → Muller), lowercase
   - Edge case: collaboration names (e.g., "TCGA Consortium") → store full name

3. **Year** — from `year = {...}` or `date = {YYYY-...}` field. Extract 4-digit year.

4. **Title snippet** — first 6 words of the `title = {...}` field (for disambiguation display)

5. **Journal/venue** — from `journal = {...}` or `booktitle = {...}` (for disambiguation)

Build an index:
```
index[last_name_lower][year] = [
  { citekey, title_snippet, journal },
  { citekey, title_snippet, journal },
  ...
]
```

Example:
```
index["smith"]["2023"] = [
  { citekey: "smith2023towards", title: "Towards multimodal...", journal: "Nature" },
  { citekey: "smith2023deep",    title: "Deep learning for...",  journal: "Cell" }
]
```

See `references/bib-formats.md` for handling unusual BibTeX formatting or special
characters in author names.

---

## Step 3 — Scan the Manuscript for Placeholders

Read the manuscript. Find every citation placeholder using these patterns:

| Pattern | Example | Interpretation |
|---|---|---|
| `[AuthorYYYY]` | `[Smith2023]` | Single citation, no suffix |
| `[AuthorYYYY_N]` | `[Smith2023_2]` | Second paper by Smith in 2023 |
| `[AuthorYYYYa]` | `[Smith2023a]` | Alternate suffix style |
| `[Author1YYYY; Author2YYYY]` | `[Smith2023; Jones2024]` | Multiple citations in one bracket |
| `[@citekey]` | `[@smith2023towards]` | Already in pandoc format — skip |

Use Grep to find all candidates:
```bash
grep -oE "\[[A-Z][a-zA-Z]+[0-9]{4}[a-z_0-9]*\]" [manuscript_path]
```

For each unique placeholder found:
- Extract: `author_part` = everything before the 4-digit year
- Extract: `year` = the 4-digit number
- Extract: `suffix` = anything after the year (empty, `_2`, `a`, `b`, etc.)
- Normalize: lowercase `author_part` for lookup

Build a list `placeholders[]` of unique placeholder strings found.

---

## Step 4 — Match Placeholders to Citekeys

For each placeholder, attempt to find its citekey in the index:

**Step 4a: Look up in index**

```
author_lower = lowercase(author_part)  # e.g., "smith"
candidates = index[author_lower][year]  # list of matching entries
```

**Step 4b: Apply suffix logic**

If `suffix` is empty or `a`: use `candidates[0]` (first match by .bib file order)
If `suffix` is `_2` or `b`: use `candidates[1]`
If `suffix` is `_3` or `c`: use `candidates[2]`
(If the indexed position doesn't exist, treat as UNMATCHED)

**Step 4c: Assign a resolution status**

| Status | Condition | Action |
|---|---|---|
| `AUTO` | Exactly 1 candidate, no suffix conflict | Replace automatically |
| `AUTO` | Multiple candidates, suffix resolves which one | Replace automatically |
| `AMBIGUOUS` | Multiple candidates, no suffix to disambiguate | Ask user |
| `UNMATCHED` | 0 candidates for this author+year | Flag for manual review |
| `SKIP` | Already in `[@...]` format | Leave untouched |

Build two lists:
- `auto_replacements[]`: `{ placeholder, citekey }` — ready to apply
- `ambiguous[]`: `{ placeholder, candidates[] }` — need user input
- `unmatched[]`: `{ placeholder, context }` — cannot resolve

---

## Step 5 — Resolve Ambiguous Cases (User Interaction)

Before writing anything, present all ambiguous cases to the user as a single batch.

Format:
```
Found N ambiguous citation(s) that need your input:

[1] Placeholder: [Wang2023]
    Option A: wang2023multimodal  — "Multimodal fusion for cancer..." (Nature Medicine)
    Option B: wang2023digital     — "Digital twin framework for..."  (npj Digital Medicine)
    → Enter A or B (or skip with S):

[2] Placeholder: [Chen2024]
    Option A: chen2024foundation  — "Foundation models in pathology" (Cell)
    Option B: chen2024clinical    — "Clinical translation of AI..."  (Lancet Digital Health)
    → Enter A or B (or skip with S):
```

Wait for the user to respond to each. Record their choices. If the user skips (S),
mark that placeholder as UNMATCHED.

If there are 0 ambiguous cases, skip this step entirely.

---

## Step 6 — Apply Replacements

Now that all resolutions are known, do a single read-modify-write pass on the manuscript.

Replacement rules:

1. Replace `[AuthorYYYY]` with `[@citekey]`
2. Replace `[AuthorYYYY_N]` with `[@citekey]`
3. For multi-cite brackets `[Smith2023; Jones2024]`:
   - Split on `;` and resolve each part individually
   - Recombine as `[@smith2023towards; @jones2024deep]`
4. Leave `[@already_formatted]` unchanged
5. Leave UNMATCHED placeholders as-is (they need manual attention)

Apply replacements to the full manuscript text, then save to the output path.

---

## Step 7 — Save the Replacement Report

Save a report alongside the output manuscript: `cite_report_YYYYMMDD.md`

Use this structure:

```markdown
# Citation Replacement Report

**Input manuscript:** [path]
**BibTeX file:** [path]
**Output manuscript:** [path]
**Date:** YYYY-MM-DD

---

## Summary

| Status | Count |
|---|---|
| Auto-replaced | N |
| User-resolved (ambiguous) | N |
| Unmatched (manual review needed) | N |
| Already in pandoc format (skipped) | N |
| **Total placeholders found** | **N** |

---

## Replacement Map

| Placeholder | → | Citekey | Method |
|---|---|---|---|
| [Smith2023] | → | [@smith2023towards] | AUTO |
| [Wang2023] | → | [@wang2023multimodal] | USER |
| [Jones2024] | → | (not replaced) | UNMATCHED |

---

## Unmatched Placeholders (Manual Action Required)

For each UNMATCHED placeholder:

### [Jones2024]
**Context in manuscript:** "...as shown by [@Smith2023] and [Jones2024], the approach..."
**Possible causes:**
- No entry with author "Jones" and year "2024" in the .bib file
- Author name spelled differently (e.g., "Jonés", "Joness")
- Paper not yet imported to the reference library

**Suggested action:** Add the paper to the .bib file, then re-run /lit-cite.

---

## BibTeX Index Summary

Total entries parsed: N
Authors with multiple 2024 entries: [list names]
```

---

## Step 8 — Report to User

After saving, print:

```
Citation replacement complete.

Output:       [output path]
Report:       [report path]

Results:
  Auto-replaced:   N
  User-resolved:   N
  Unmatched:       N  ← see report for manual action
  Skipped:         N  (already in pandoc format)

[If N_unmatched > 0:]
  N placeholder(s) could not be matched. Add them to the .bib file
  and re-run /lit-cite to complete the replacement.
```

---

## Constraints

- **Never overwrite the source manuscript.** Always save to a new file.
- **Never guess a citekey.** If a placeholder is UNMATCHED, leave it in the output
  file as-is. A wrong citekey is worse than a placeholder.
- **Preserve surrounding punctuation.** If the manuscript has `[Smith2023].` (period
  after the bracket), keep the period in place: `[@smith2023towards].`
- **Multi-cite order.** When converting `[Smith2023; Jones2024]`, preserve the original
  order of authors in the output: `[@smith2023; @jones2024]`.
- **Do not touch prose.** Only scan text that matches the placeholder pattern. Do not
  attempt to replace author names appearing in running text (e.g., "Smith et al. found...").
