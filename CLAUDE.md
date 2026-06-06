# CLAUDE.md — Narrative Literature Review Project Instructions

> Claude Code reads this file automatically at the start of every session.
> Keep it updated as the project evolves.

---

## Project Identity

**Title (working):** Multimodal Digital Twins for Cancer: Integrating Pathology, Radiology, and Multi-Omics AI

**Type:** Narrative literature review (original review article)

> **Important:** This is a *narrative* review, NOT a systematic review. See Writing Style Guidelines for the prose conventions that follow from this distinction.

**Target journals (in priority order):**
- [npj Digital Medicine] TBD — update once confirmed

**Target length:** ~8,000–12,000 words (main text, excluding references)

**Intended audience:** Computational oncology researchers, clinical AI developers

---

## Directory Layout

```
literature-review/
├── CLAUDE.md               ← YOU ARE HERE (always read first)
├── PROMPTS.md              ← reusable prompt snippets for common tasks
├── search/                 ← raw search outputs (CSV/JSON from APIs)
├── screening/              ← ai_screened.csv · borderline_review.csv · final_screened.csv
├── pdfs/                   ← full-text PDFs + pdf_map.csv (PMID/DOI → filename mapping)
├── extractions/            ← one .md file per paper, structured notes
├── synthesis/              ← thematic clusters, gap analysis, tables
├── outline/                ← outline drafts (v1, v2, …)
├── draft/                  ← section drafts (one file per section)
├── references/             ← .bib export from Zotero, final reference list
└── scripts/                ← Python scripts for automation
```

**Rule:** Never write files outside this structure without asking first.

---

## Zotero Integration

- **Import method:** NBIB bulk import (not Zotero API) — run `scripts/export_nbib.py`, then Zotero → File → Import → select `search/included_papers.nbib`
- **Note:** Zotero auto-creates a collection named after the file (`included_papers`); rename it to `DigitalTwins-Cancer-Review` after import

When adding papers to Zotero, always:
1. Import via NBIB to get complete metadata (volume, issue, pages, MeSH terms)
2. After import, rename the auto-created collection to `DigitalTwins-Cancer-Review`
3. Log the Zotero item key in the extraction note

### Citation workflow (drafting → Word)

- **Plugin requirement:** Zotero must have **Better BibTeX** installed. It provides
  stable, reproducible citekeys and exports `references/library.bib`.
- **Drafting:** sections use `[FirstAuthorYear]` placeholders (Phase 5).
- **Phase 7a:** placeholders are replaced with pandoc citation format — `[@citekey]`
  for single, `[@key1; @key2]` for multiple. Output to `manuscript/manuscript_cited.md`
  with YAML metadata `zotero: {client: zotero, csl-style: vancouver}`.
- **Phase 7b:** run pandoc with the BBT lua filter (**Zotero must be open**):
  `pandoc -s --lua-filter=zotero.lua manuscript_cited.md -o manuscript.docx`
  The filter contacts Zotero's JSON-RPC API and embeds live citation fields into the docx.
  `manuscript/zotero.lua` is the filter file (downloaded from retorque.re).
- **Phase 7c (manual):** Open `manuscript.docx` in Word → Zotero toolbar →
  Document Preferences → select citation style → OK. Then click
  "Add/Edit Bibliography" to insert the formatted reference list.
- **Do NOT** use pandoc `--citeproc` or `--bibliography`: they produce static text that
  the Zotero Word plugin cannot edit afterward.

---

## Literature Scope

### In-scope topics
- Cancer digital twins (any organ/cancer type)
- Multimodal AI integrating ≥2 of: pathology (WSI), radiology (CT/MRI/PET), multi-omics (genomics/transcriptomics/proteomics/metabolomics)
- Foundation models applied to oncology multimodal data
- Computational pathology + imaging integration
- Treatment response prediction using multimodal data
- Digital twin frameworks for treatment simulation/personalization

### Out-of-scope
- Single-modality studies (unless methodologically landmark)
- Digital twins in non-cancer disease (unless directly transferable)
- Clinical trial design papers without AI/computational component
- Pure omics studies without imaging integration

### Date range
- Primary focus: 2019–2026
- Landmark earlier papers: include if foundational

---

## Writing Style Guidelines

- **Voice:** Third-person academic, active where possible ("We argue…" only in Discussion/Conclusion)
- **Tone:** Authoritative but balanced; acknowledge limitations and open questions
- **Structure:** Narrative synthesis, not annotated bibliography — papers should serve arguments, not the reverse
- **Citation style:** Use `[FirstAuthorYear]` placeholders during drafting (e.g., `[Chen2023]`); final replacement done in Phase 6
- **Avoid:** Bullet-point lists in the main text; starting consecutive sentences with the same subject; vague openers like "Recently, …" or "In recent years, …"
- **Em-dash (—) policy:** Use sparingly — no more than 8–10 instances per 10,000 words. Reserve exclusively for a sharp, unavoidable interruption or appositive that a comma or parenthesis would weaken. Do NOT use em-dashes to attach subordinate clauses, list examples, or replace colons. Prefer commas, parentheses, or a new sentence instead. Before drafting or polishing, count existing em-dashes; if the count exceeds the threshold, revise those sentences first.

---

## Narrative vs. Systematic Review: Prose Conventions

This is a **narrative literature review**. Prose must not read like a systematic review. The distinction matters because narrative reviews synthesize arguments; systematic reviews enumerate evidence.

### Prohibited (systematic review language)

| Pattern | Example | Why prohibited |
|---|---|---|
| Exact paper counts in prose | "89 studies integrate pathology with genomics" | Implies exhaustive enumeration; SLR convention |
| Exhaustive-corpus statements | "all 280 papers in the corpus", "None of the 280 reviewed studies" | Signals a PRISMA-style corpus audit |
| Meta-sentences about the review | "This review synthesizes 280 studies published between 2019 and 2026" | Abstract-of-a-SLR phrasing |
| "in this corpus" / "the corpus" as a technical term | "the most sparsely represented integration in the corpus" | Implies a formally bounded, screened corpus |

### Preferred (narrative review language)

- **Qualitative scope descriptors:** "the dominant evidence base", "a small but growing frontier", "relatively few published studies", "the majority of papers"
- **Literature reference:** "the reviewed literature", "the published literature", "published work in this area", "studies examined here"
- **Gap language:** "prospective validation has not been reported", "no published study documents…", "this integration remains unexplored"
- **Evidence sufficiency:** Cite 2–4 representative, high-quality exemplars to support a claim; do not attempt to list all studies that support it

### Acceptable uses of numbers

Numbers are permitted when they come from a *specific cited study* (patient counts, AUC values, cohort sizes). They are not permitted when they describe the review's own paper collection.

✓ "validated on 2,279 patients across 12 centers [Huang2025_2]"  
✗ "the 58 radiogenomics papers in this review's corpus"

---

## Screening CSV Format

`screening/ai_screened.csv` (written by `scripts/batch_screen.py`) has these columns:

| Column | Description |
|---|---|
| PMID | PubMed ID |
| Title | Article title |
| Year | Publication year |
| Journal | Journal name |
| DOI | DOI |
| Score | 2 = Include · 1 = Borderline · 0 = Exclude |
| Reason | ≤15-word AI justification |

`screening/borderline_review.csv` — Score=1 papers for manual triage; columns: PMID, Title, Year, Journal, Abstract, DOI, Reason, Include (fill 1=include / 0=exclude).  
`screening/final_screened.csv` — Same columns as `ai_screened.csv`; Score=1 rows resolved to 0 or 2 after manual review.

---

## Extraction Note Format

Each file in `extractions/` should follow this template exactly:

```markdown
# [Short title or FirstAuthorYear]

## Metadata
- **Full title:**
- **Authors:**
- **Journal / Conference:**
- **Year:**
- **DOI:**
- **Zotero key:**
- **Paper type:** (Original research / Review / Perspective / Other)
- **Text source:** (Full text PDF / Abstract only)
- **Modalities covered:** (e.g., Pathology + Radiology / Omics + Radiology)
- **Cancer type(s):**
- **In-scope score:** (1 = peripheral, 2 = relevant, 3 = core)

## Methods
- **Framework / Architecture:**
- **Key techniques:**
- **Datasets used:**
- **Validation approach:**

## Key Findings
(2–4 bullet points)

## Limitations (as stated or implied)

## Relevance to This Review
- **Which section(s) it feeds:**
- **Key claim to cite:**
- **Any conflicting findings with other papers:**
```

---

## Python Environment

This project uses **uv** for reproducible dependency management.

- **Run any script:** `uv run python scripts/xxx.py`
- **Add a new package:** `uv add <package-name>`  (updates `pyproject.toml` + `uv.lock` automatically)
- **Reproduce environment on a new machine:** `uv sync`
- **Never use** `pip install` directly — always go through `uv add`

The virtual environment lives in `.venv/` (gitignored). `uv.lock` is committed to git and guarantees exact reproducibility.

---

## Session Startup Checklist

At the beginning of each Claude Code session, do the following:
1. Read `CLAUDE.md` (this file)
2. Read `outline/outline_current.md` if it exists
3. Briefly state: current phase, what was done last session (check `LOG.md`), and what we're doing today
4. Ask the user to confirm before starting any destructive actions (overwriting files, deleting entries)

---

## LOG.md Convention

Append a brief entry to `LOG.md` at the end of each session:

```
## YYYY-MM-DD
- Phase: X
- Done: [what was completed]
- Next: [what to do next session]
```
