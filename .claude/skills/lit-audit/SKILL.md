---
name: lit-audit
description: >
  Factual fidelity audit for academic literature review manuscripts. Cross-references
  every specific quantitative claim in the manuscript — metric values (AUC, accuracy,
  F1, BLEU, hazard ratio, etc.), dataset names, model/architecture names, and sample
  sizes — against the source extraction notes or paper summaries, then reports which
  claims are verified, which are mismatched, and which lack a traceable citation.
  Use this skill whenever a manuscript draft has been updated, before any style-polish
  pass, or any time you suspect numbers may have drifted from their source notes.
  Works with any markdown manuscript + extraction note workflow, regardless of domain
  or citation style. Invoke with /lit-audit.
allowed-tools: Read Glob Grep Write Bash
metadata:
  version: "2.0"
---

# Lit-Audit — Factual Fidelity Auditor

Your job is **verification, not rewriting**. You read the manuscript, read the
extraction notes, and check whether specific claims in the manuscript can be traced
to the notes. You do not edit the manuscript. You produce an audit report and stop.

Read actual files for every claim. Do not rely on memory.

---

## Step 1 — Discover the Project Layout

You need three paths before you can proceed: the manuscript file, the extraction notes
directory, and a location to save the report. Try to auto-detect them.

### 1a. Locate the manuscript

Check these locations in order, stopping at the first match:

1. A path the user explicitly named in their message
2. `manuscript/*.md` (any .md file in a manuscript/ directory)
3. `draft/*.md`
4. `output/*.md`
5. Any `.md` file in the current directory that looks like a full manuscript (> 3,000
   words, contains `## Introduction` or `## Abstract`)

If you find exactly one candidate, proceed. If you find multiple, name them and ask
the user which one to audit before continuing.

### 1b. Locate the extraction notes directory

Check these locations in order, stopping at the first match with ≥ 3 `.md` files:

1. `extractions/`
2. `notes/`
3. `papers/`
4. `summaries/`
5. `readings/`
6. `paper_notes/`

If none of these exist, ask the user where the extraction notes live.

### 1c. Choose an output location for the report

Use the first of these that exists as a directory:

1. `synthesis/`
2. `output/`
3. `reports/`
4. Same directory as the manuscript

### 1d. State the discovered layout

Before proceeding, print one line to the user:

```
Auditing: [manuscript path]
Extractions: [N files in extraction dir]
Report will be saved to: [output dir]/audit_YYYYMMDD.md
```

---

## Step 2 — Build the Extraction Index

Use Glob to list all `.md` files in the extraction notes directory. For each file,
read it and record the following into an in-memory index, keyed by the **filename stem**
(e.g., `Smith2023`, `Chen2024_2`):

| What to capture | Where to look (try all of these) |
|---|---|
| Quantitative metrics and values | Any section header containing "finding", "result", "metric", "quantitative", "evidence", or "performance"; also any line with a decimal number followed by % or a metric keyword |
| Dataset names | Lines containing "dataset", "data used", "corpus", "benchmark", "cohort" |
| Model / framework / architecture names | Lines containing "model", "framework", "architecture", "method", "system", "approach" with a proper-noun name nearby |
| Sample / patient / item counts | Lines with patterns like "N=", "n=", "X patients", "X subjects", "X samples", "X images", "X documents" |

**Handling different extraction formats:** Some projects use strict templates with
labeled fields (e.g., `**Datasets used:**`, `**Framework / Architecture:**`). Others
use free-form prose notes. Search for both. When in doubt, collect every line that
contains a number, a proper noun, or a named entity — false positives in the index
are harmless; false negatives cause missed mismatches.

See `references/claim-patterns.md` for the full taxonomy of what counts as auditable
and how to extract it from notes with different structures.

---

## Step 3 — Scan the Manuscript for Auditable Claims

Read the manuscript section by section. For each paragraph, identify every
**auditable claim**: a claim that states something specific and traceable.

Auditable claim types:

- A **metric value**: a number attached to a named performance measure
  (AUC, accuracy, F1, BLEU, ROUGE, perplexity, C-index, hazard ratio, sensitivity,
  specificity, RMSE, R², error rate, etc.)
- A **named dataset**: a specific proper-noun dataset name
- A **named model or system**: a specific model, framework, or architecture name
  that authors coined (not a generic category like "a transformer")
- A **sample count**: a specific number of patients, subjects, samples, documents,
  images, or items tied to a named study
- A **study-level result**: a quantified finding attributed to a specific cited study

For each auditable claim, record:
- `claim_text`: verbatim excerpt (≤ 30 words)
- `citation_key`: the citation immediately adjacent to this claim, if any
- `section`: the heading the claim appears under

Build a list `claims[]` of `{claim_text, citation_key, section}`.

Claims with no number AND no named proper noun are not auditable. Skip them.

---

## Step 4 — Cross-Reference Claims Against the Extraction Index

For each item in `claims[]`:

**4a. Identify the source extraction file.**

The citation key in the manuscript should correspond to a filename stem in the
extraction directory. Try these mapping strategies:

1. Direct match: citation key `Smith2023` → `Smith2023.md`
2. Suffix match: `[@Smith2023]` or `[Smith2023]` → strip brackets and `@`
3. Disambiguation: if multiple files share a root (e.g., `Smith2023`, `Smith2023_2`),
   check all of them; VERIFIED if found in any
4. Numeric citations (`[1]`, `[23]`): if the project uses numbered references, look
   for a `references/` or `bibliography/` file that maps numbers to paper identifiers,
   then map to extraction files from there
5. No citation key present → mark status as `UNCITED`

**4b. Check the claim against the extraction.**

Search the indexed content from Step 2 for the specific value, name, or count in the
claim. Apply these matching rules:

- Metric values: match within ±0.01 for 2-decimal numbers; within ±0.5% for percentages
- Dataset names: case-insensitive match; acronym expansions count (e.g., "IMDB" = "Internet Movie Database")
- Model names: case-insensitive match; acronyms and spelled-out forms count
- Sample counts: exact match required for N ≥ 100; tolerance of ±5 for N < 100

**4c. Assign a status.**

| Status | Meaning |
|---|---|
| `VERIFIED` | Value or name found in the extraction note |
| `APPROX` | Value present but differs by rounding or minor formatting (flag, low priority) |
| `MISMATCH` | A different value appears in the extraction for the same metric/item |
| `NOT_IN_EXTRACTION` | Extraction file exists but the specific claim is absent from it |
| `UNCITED` | No citation key is adjacent to this specific claim |
| `NO_EXTRACTION` | No extraction file found matching the citation key |

---

## Step 5 — Write the Audit Report

Save the report as `audit_YYYYMMDD.md` in the output directory identified in Step 1.

Use this structure exactly:

```markdown
# Factual Fidelity Audit

**Manuscript audited:** [path/filename]
**Extraction notes:** [N files in extraction dir path]
**Date:** YYYY-MM-DD
**Total auditable claims found:** N
**Verified:** N  |  Approx: N  |  Mismatch: N  |  Not in extraction: N  |  Uncited: N  |  No extraction: N

---

## Summary

[2–3 sentences covering: overall fidelity level (% verified), which sections carry
the highest risk, and the recommended next step — e.g., whether to fix mismatches
before proceeding, or whether the manuscript is clean enough to move to style editing.]

---

## Audit Table

| # | Section | Claim (verbatim excerpt) | Citation key | Status | Notes |
|---|---|---|---|---|---|
| 1 | Introduction | "AUC of 0.91 on external..." | [@Smith2023] | VERIFIED | Found in Key Findings bullet 2 |
| 2 | Section 3.2 | "1,250 patients..." | [@Jones2024] | MISMATCH | Note says 924 patients |
| ... | | | | | |

---

## Mismatches Requiring Correction

For each MISMATCH, NOT_IN_EXTRACTION, or NO_EXTRACTION row:

### Claim #N — [STATUS]
**Manuscript text:** "[exact quote]"
**Citation:** [key]
**Found in extraction:** "[relevant text from extraction note, or 'not found']"
**Suggested action:** [correct the number / remove the claim / add a qualifying caveat]

---

## Uncited Claims

[List each UNCITED claim. These carry hallucination risk independent of whether the
value is numerically correct, because there is no traceable source.]

---

## Rounding Discrepancies (APPROX — lower priority)

[List APPROX cases. These do not block the next editing pass but should be standardized
before final submission.]
```

---

## Step 6 — Report to the User

After saving, print this summary (fill in actual values):

```
Audit complete → [output path]

Claims audited:   N
  Verified:       N  (X%)
  Mismatch:       N  ← fix before style editing
  Not in notes:   N  ← fix before style editing
  Uncited:        N  ← review and cite or remove
  No note found:  N
  Rounding only:  N  (low priority)

Highest-risk section: [section name with most mismatches/uncited]
```

Do not list individual mismatches in the terminal — they are in the saved report.

---

## Constraints

- **Do not edit the manuscript.** Your only output is the audit report file.
- **Do not paraphrase extraction notes.** Quote exact text when citing evidence.
- **Do not infer correctness.** If a claim is absent from the extraction note, the
  status is NOT_IN_EXTRACTION — never assume the note is simply incomplete.
- **Skip non-specific sentences.** Audit only claims with a specific number, named
  dataset, named model, or named sample count. Skip all qualitative generalizations.
- **If context runs out mid-audit**, save a partial report with a note:
  `## INCOMPLETE — last extraction processed: [filename]`
  so the user knows where to resume.
