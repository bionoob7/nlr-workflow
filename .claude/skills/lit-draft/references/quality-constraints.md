# Quality Constraints Reference

Load this when you need exact threshold values, edge-case rules, or worked examples
of constraint violations and their rewrites.

The thresholds in this file are project-agnostic. Word budgets are always read from
the current outline — never from this file. This file governs the *quality rules*
applied to whatever budget the outline specifies.

---

## 1. How to Read the Word Budget from the Outline

Do not use a hardcoded table. Read the budget from the outline entry for the target
section each time. Look for these patterns:

```
Target word count: 1,200 words
~1,200 words
budget: 1,200
(1,200 words)
[approx. 1,200 words]
```

If no budget is stated in the outline, apply these defaults based on section role:

| Section role | Default budget range |
|---|---|
| Introduction (no subsections) | 900–1,200 words |
| Conclusion (no subsections) | 600–900 words |
| Short conceptual section | 600–900 words |
| Standard body section | 1,000–1,500 words |
| Long technical body section | 1,500–2,500 words |
| Subsection within a larger section | 400–900 words |

The hard ceiling is always **budget × 1.10**. A section drafted at budget × 1.12 must
be tightened before saving — not accepted as "close enough."

---

## 2. Citation Density

**Target:** ≥ 1 citation per 60–80 words of body text in technical sections.
Introduction and conclusion have looser density but should still cite specific papers
for any empirical claim.

**Counting method:** Count unique `[FirstAuthorYear]` bracket occurrences divided by
total word count. A multi-cite bracket `[Smith2023; Jones2024]` counts as 2 citations.

### What requires a citation

- Any claim about what a specific study found, used, or concluded
- Any named metric, dataset, patient count, or model name attributed to a study
- Comparative statements ("X outperforms Y", "only N papers have...") unless both
  sides are already established in the same paragraph
- Any summary generalization about "most studies" or "the majority of papers"

### What does not require a citation

- Definitional statements about established methods or concepts
  ("BERT uses a transformer encoder pre-trained on masked language modeling")
- Logical connectives between already-cited claims in the same paragraph
- Statements about what *this review* argues or covers

### Citation format

- Single: `[Smith2023]`
- Multiple in one bracket: `[Smith2023; Jones2024]` (semicolon-separated)
- These placeholders are converted to pandoc `[@citekey]` format in a later workflow
  step — do not pre-convert them during drafting

### Citation placement rule

Placeholders become numbered superscripts or parenthetical markers when typeset.
A sentence must never open with a placeholder, because that produces a citation
marker as the first character.

**When the author is named in prose:** write the name out; put the placeholder at
the end of the clause.
- ✓ "Smith et al. achieved a C-index of 0.81 on TCGA-BRCA (N=892) [Smith2023]."
- ✓ "Jones and Wang report that the model generalizes across institutions [Jones2024]."

**When the author is not named:** cite inline after the claim, before the period.
- ✓ "Cross-institution generalization is the most consistently reported limitation
  [Smith2023; Jones2024]."

**Never:**
- ✗ "[Smith2023] achieved a C-index of 0.81..." — placeholder opens the sentence
- ✗ "[Jones2024] report that..." — placeholder as the grammatical subject

---

## 3. Numeric Accuracy

**Rule:** Every specific number in the draft must be traceable to a line in an
extraction file. If it is not there, write the qualitative claim instead.

### How to read an extraction file

Extraction file formats vary by project. In Step 3, you read a sample file to
discover the format. Common fields that carry verifiable numbers:

| Information type | Common field names to look for |
|---|---|
| Model / architecture name | "Framework", "Architecture", "Model", "System" |
| Primary metric | "Key metrics", "Performance", "Results", "AUC", "C-index" |
| Dataset name | "Datasets", "Data", "Cohort", "Benchmark" |
| Patient count | "N =", "n patients", "cohort size", "n=" |
| Validation type | "Validation", "External", "Prospective", "Internal" |

If the extraction uses a structured template (e.g., `## Key Findings` with bullet
points), the bullet text is reliable for qualitative claims but the specific numbers
should still be matched to a dedicated metrics field.

### Hard vs. soft numbers

**Hard (must be in extraction before writing):**
AUC, C-index, accuracy, F1, AUROC, RMSE, hazard ratio, p-value, sensitivity,
specificity, patient count (N=...), cohort name, model/architecture name.

**Soft (qualitative claim is always safe):**
"superior performance", "significantly improved", "outperformed the unimodal
baseline", "achieved state-of-the-art results on this benchmark" — these are
acceptable if the direction of the finding is confirmed in the extraction's Key
Findings section.

### Worked examples

**Correct — value confirmed in extraction:**
> Extraction: `Key metrics: C-index · 0.74 · TCGA-BLCA · N=412`
> Draft: "Chen et al. achieved a C-index of 0.74 on TCGA-BLCA (N=412) [Chen2023]."

**Correct — value absent from extraction:**
> Extraction: Key metrics field empty; Key Findings: "outperformed unimodal baseline"
> Draft: "Chen et al. demonstrated improved survival discrimination on TCGA-BLCA
> relative to the unimodal baseline [Chen2023]."

**Incorrect — value from model memory:**
> Extraction: no metric value found
> Draft: "Chen et al. achieved a C-index of 0.74 [Chen2023]." ← fabricated

**Incorrect — placeholder as sentence opener:**
> Draft: "[Chen2023] achieved a C-index of 0.74..." ← citation marker opens sentence

---

## 4. Em-Dash Policy

**Limit:** ≤ 1 em-dash (—) per 1,000 words of the section. Compute the limit from
the section's word budget:

```
em_dash_limit = ceil(budget / 1000)

Examples:
  800-word section  → limit = 1
  1,200-word section → limit = 2
  2,000-word section → limit = 2
  2,500-word section → limit = 3
```

**Count em-dashes in the saved file:**
```bash
grep -o "—" draft/section_slug.md | wc -l
```

Note: Some manuscripts use `--` as an em-dash surrogate. Count both if the project
uses `--` consistently.

### Permitted use (one instance maximum)

A sharp appositive or interruption where a comma would genuinely weaken the sentence:
> "The third approach — contrastive pretraining — aligns representations without
> requiring paired training labels."

### Prohibited uses (rewrite every instance)

| Prohibited pattern | Rewrite |
|---|---|
| Attaching a subordinate clause: "The model performed well — even on external cohorts." | "The model performed well, generalizing to external cohorts." |
| Introducing examples: "Three modalities were integrated — pathology, radiology, and omics." | "Three modalities were integrated: pathology, radiology, and omics." |
| Replacing a colon: "The result was clear — multimodal models outperform unimodal." | "The result was clear: multimodal models consistently outperformed their unimodal counterparts." |
| Attaching a cause: "Generalization failed — the dataset was too small." | "Generalization failed because the dataset was too small." |

---

## 5. Register Self-Check

### 5a — Antithesis constructions (max 1 per section)

**Detection — grep for the pattern:**
```bash
grep -iE "\bnot\b.{5,80}\bbut\b" draft/section_slug.md | wc -l
```

Also manually scan for "rather than X, Y" when used rhetorically.

**Max allowed:** 1 per section. If the count exceeds 1, rewrite extras as plain
declaratives — state the conclusion directly without framing it as a negation.

| Before | After |
|---|---|
| "not prediction from correlates but simulation from causes" | "genuine counterfactual simulation of tumor response" |
| "not a minor limitation but a fundamental gap" | "a fundamental limitation that current methods have not resolved" |
| "not just powerful but clinically deployable" | "clinically deployable, with demonstrated generalization across institutions" |

### 5b — Editorializing adjectives (target 0 prohibited; ≤3 controlled)

**Prohibited words (0 instances in the final draft):**
striking, remarkable, unprecedented, groundbreaking, particularly consequential,
not a minor caveat, precisely the, crucially (as sentence opener), compellingly,
transformative (applied to a specific result).

**Controlled words (≤3 total for the section — not per paragraph):**
important, notable, significant (non-statistical sense), powerful, key, compelling.

**Grep to count prohibited words:**
```bash
grep -iEo "\b(striking|remarkable|unprecedented|groundbreaking|compellingly|transformative)\b" draft/section_slug.md | wc -l
```

**Rewrite strategy:** Delete the editorial frame; the specific fact carries the weight.

| Before | After |
|---|---|
| "The striking finding that AUC reached 0.94 across six cohorts..." | "The model achieved AUC=0.94 across six independent cohorts..." |
| "a particularly consequential gap is the absence of..." | "the absence of... represents the primary barrier to clinical deployment" |
| "this is a remarkable achievement given the dataset size" | "this result was achieved on a cohort of [N] patients" |

---

## 6. Prose Quality Checks (non-quantifiable)

These do not produce a count but should be checked by reading the draft once before
saving.

**No consecutive same-subject sentences:** If two adjacent sentences both open with
"The model", "This approach", "These results", or any repeated grammatical structure,
rewrite one to vary the entry point. Options: open with a participle clause, a
prepositional phrase, a subordinate clause, or the object instead of the subject.

**No vague openers:** Delete sentences that open with:
- "Recently, ..." / "In recent years, ..."
- "It is worth noting that..."
- "It should be noted that..."
- "As is well known..."
Restate the content more directly, or fold it into the preceding sentence.

**Transition sentences must name something specific:** A closing transition that says
"The next section discusses [topic]" without naming a specific gap, finding, or
unanswered question should be rewritten. The transition should advance the argument:

  ✗ "The next section examines radiology-based approaches."
  ✓ "These pathology-centric frameworks treat radiology as a secondary validation
    tool; the radiogenomics literature that follows asks whether imaging alone can
    expose molecular state, making biopsy unnecessary for certain clinical tasks."

---

## 7. Section Slug Algorithm (project-agnostic)

Generate slugs deterministically from the section title so draft files sort in
outline order and are easy to find.

### Algorithm

1. Extract the section number from the heading if present.
   - "3." → `03`
   - "3.1" → `03_1`
   - "3.1.2" → `03_1_2`
   - No number → skip the number prefix
2. Extract the first 3–4 *meaningful* words of the title. Drop: articles (a, an, the),
   prepositions (of, in, for, with, and, or, to, from, by), conjunctions.
3. Lowercase; replace spaces and all punctuation with underscores; collapse multiple
   underscores.
4. Combine: `[number_prefix]_[short_title].md`

### Examples

| Full section title | Slug |
|---|---|
| 1. Introduction | `01_introduction.md` |
| 2. A Taxonomy of Multimodal Integration | `02_taxonomy_multimodal.md` |
| 2.1 — The Complementarity Argument | `02_1_complementarity.md` |
| 3. Pathology-Centric Multimodal Integration | `03_pathology_centric.md` |
| 3.2 — Spatial Transcriptomics and Tissue Architecture | `03_2_spatial_transcriptomics.md` |
| 4. Radiogenomics and Radiology-Omics Integration | `04_radiogenomics.md` |
| 5.2 — Cancer Digital Twin Frameworks | `05_2_digital_twins.md` |
| 6. Challenges, Gaps, and Future Directions | `06_challenges.md` |
| 7. Conclusion | `07_conclusion.md` |
| Methods (no number) | `methods.md` |
| Related Work (no number) | `related_work.md` |
| Background and Motivation | `background_motivation.md` |

### Edge cases

- **Title with special characters** (`—`, `:`, `/`): treat as word boundaries, then
  drop: "Section 3 — Digital Twins: Current State" → `03_digital_twins_current.md`
- **Duplicate short titles across sections**: append the full section number:
  `02_methods.md` vs `05_methods.md`
- **Subsection with no parent number in the identifier**: prefix with parent section
  number from the outline: if §3 has three subsections and the user asks for §3.2,
  the slug is `03_2_[title].md`

---

## 8. Narrative vs. Systematic Review Language (NLR projects)

This section applies only when CLAUDE.md identifies the project as a **narrative
literature review**. Skip this section entirely for systematic reviews or scoping
reviews.

The core distinction: a narrative review synthesizes arguments; a systematic review
enumerates evidence. Four prose patterns cross the genre line and must not appear in
NLR drafts.

---

### 8a — Paper-count claims about the review corpus (target: 0)

These sentences count the review's own paper collection as if reporting a database
query result. They signal SLR posture.

**Detection:** any number followed by a paper-class noun tied to the review's own
collection:

| Pattern | Example |
|---|---|
| `[N] papers in this corpus` | "89 papers in this corpus integrate pathology with omics" |
| `[N] studies in this review` | "the 12 studies in this review that attempt fusion" |
| `only [N] of the [N] reviewed papers` | "only 12 of the 280 reviewed papers" |
| `the [N] [adjective] papers` (when referring to a sub-cluster) | "the 30 digital twin papers in the corpus" |

**Rewrite — qualitative scope descriptors:**

| Before | After |
|---|---|
| "89 papers in this corpus integrate pathology with genomics" | "the dominant evidence base integrates pathology with genomics" |
| "only 12 of the 280 reviewed papers attempt three-modality fusion" | "relatively few published studies attempt three-modality fusion" |
| "the 30 digital twin papers in this corpus" | "digital twin papers in the reviewed literature" |
| "the 58 radiogenomics studies reviewed here" | "the radiogenomics literature examined here" |

**What counts as the review's paper collection vs. a cited study:**

✓ "validated on 2,279 patients across 12 centers [Huang2025]" — this is a number
  from a cited study; always permitted.
✗ "89 studies in this corpus integrate pathology with omics" — this is a number
  describing the review's own paper set; always rewrite.

---

### 8b — Exhaustive-corpus assertions (target: 0)

These sentences assert that a finding holds across an entire bounded paper set,
signaling that a systematic search was conducted and audited.

**Detection:**

| Pattern | Example |
|---|---|
| `none of the [N] reviewed studies` | "none of the 280 reviewed studies reports…" |
| `absent from all [N] papers` | "prospective validation is absent from all 280 papers" |
| `across all [N] reviewed` | "absent across all 89 pathomic papers reviewed" |

**Rewrite — field-level gap language:**

| Before | After |
|---|---|
| "None of the 280 reviewed studies reports a prospective result" | "The reviewed literature contains no prospective results" |
| "Prospective validation is absent from all 280 papers in the corpus" | "Prospective validation is absent from the reviewed literature" |
| "no study in the corpus documents a change in patient management" | "no published study documents a change in patient management" |
| "federated learning is entirely absent from this corpus" | "federated learning has not been reported in the published literature" |

The gap remains accurately stated; only the corpus-boundary framing is removed.

---

### 8c — Meta-sentences about the review's scope (target: 0)

Sentences that describe the paper collection as a numbered dataset belong in a
methods section of a systematic review, not in the body of a narrative review.

**Detection:**

| Pattern | Example |
|---|---|
| `this review synthesizes [N] studies` | "This review synthesizes 280 studies published between 2019 and 2026" |
| `this section reviews [N] papers` | "This section reviews 45 radiogenomics papers" |
| Opening with the paper count as the subject | "The 89 pathomic fusion papers reviewed here show…" |

**Rewrite — scope description without the count:**

| Before | After |
|---|---|
| "This review synthesizes 280 studies published between 2019 and 2026 that integrate at least two modalities" | "The studies surveyed here span the intersection of whole-slide histopathology, cross-sectional radiology, and molecular omics, integrating at least two of these modality classes within a single analytical framework." |
| "The 89 pathomic fusion papers reviewed here map four architectural strategies" | "Pathomic fusion papers map four broad architectural strategies" |
| "This section reviews 45 radiogenomics papers" | "Radiogenomics has produced a substantial body of work spanning…" |

---

### 8d — "corpus" as a formal SLR boundary term (target: ≤ 2)

"The corpus" and "in this corpus" are acceptable twice in a manuscript — to avoid
awkward repetition of "the reviewed literature." Every instance beyond two should
be replaced.

**Permitted replacements:**
- "the reviewed literature"
- "published work in this area"
- "the studies examined here"
- "the published literature"
- "among the studies surveyed"

**Exception:** "corpus" is always kept when it refers to a text corpus in the NLP
sense, or appears in a quoted paper title or dataset name.

---

### 8e — Permitted uses of numbers (never rewrite these)

Numbers are always permitted when they originate from a *cited study*:

✓ "validated on 2,279 patients across 12 centers [Huang2025]"
✓ "AUC of 0.91 on an independent cohort of 847 patients [Chen2024]"
✓ "across four cancer types and three institutions [Wang2023]"

Numbers used to enumerate the review's own *arguments* (not papers) are also fine:

✓ "Three barriers limit clinical translation" — enumerates arguments, not papers
✓ "Two architectural families dominate the pathomic literature" — qualitative claim
   about patterns in the field, not a count of the review's paper collection
