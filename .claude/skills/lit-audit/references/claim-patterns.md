# Claim Patterns — Audit Taxonomy and Detection Heuristics

Load this reference when a claim type is ambiguous or when deciding whether a given
sentence is auditable. This document is domain-agnostic and applies to any field where
a manuscript cites specific study results.

---

## 1. Metric Value Claims

### What they are
A sentence that states a named performance metric followed by a specific number.

### Examples across domains

**Machine learning / AI:**
- "achieved an AUC of 0.91 on the held-out test set"
- "F1 score of 0.84, outperforming the baseline at 0.71"
- "BLEU-4 score of 38.2 on the WMT14 benchmark"
- "top-1 accuracy of 85.3% on ImageNet"
- "perplexity of 14.7 on the Penn Treebank test set"

**Clinical / biomedical:**
- "C-index of 0.74 (95% CI: 0.69–0.79)"
- "sensitivity of 88.3% and specificity of 91.2%"
- "hazard ratio of 1.87 (p < 0.001)"
- "overall survival at 5 years of 63%"

**Social science / NLP:**
- "Cohen's κ of 0.82 for inter-rater agreement"
- "Pearson r = 0.67 between model scores and human judgments"
- "mean absolute error of 2.3 points on the Likert scale"

### How to verify
Search the extraction note for the metric name. The number must appear there.
Synonym equivalences that count as VERIFIED:
- "concordance index" = "C-index"
- "area under the curve" = "AUC"
- "receiver operating characteristic" may precede the AUC value

If the extraction gives a confidence interval and the manuscript states only the point
estimate, mark APPROX (not MISMATCH).

### Do NOT audit
- "outperformed existing methods" (no number)
- "achieved high accuracy" (no number)
- "showed statistically significant improvement" (no effect size stated)

---

## 2. Dataset / Benchmark Name Claims

### What they are
A sentence that names a specific dataset, corpus, or benchmark by its proper noun.

### Examples across domains

**Computer vision / AI:** ImageNet, COCO, CIFAR-10, SQuAD, GLUE, SuperGLUE,
MS-COCO, Open Images, Kinetics-400

**Clinical / biomedical:** TCGA, UK Biobank, MIMIC-IV, PhysioNet, NLST,
CAMELYON16, TCIA, eICU Collaborative Research Database

**NLP:** Penn Treebank, WMT14, MultiNLI, BookCorpus, Common Crawl

**Social science:** General Social Survey, ANES, World Values Survey

### How to verify
The dataset name must appear in the extraction note — typically under a field labeled
"datasets", "data", "corpus", "benchmark", or "evaluation" — or embedded in the
findings text.

### Edge cases
- A partial name like "a TCGA cohort" is less specific than "TCGA-BRCA"; audit only
  the latter (it identifies a specific subset).
- "a proprietary internal dataset" with no identifier is not auditable.
- Version numbers matter: "COCO 2017" vs. "COCO 2014" are different datasets.

---

## 3. Model / System / Architecture Name Claims

### What they are
A sentence that names a specific model, system, or architecture by the proper noun
the authors gave it — not a generic category.

### Examples across domains

**Named models:** GPT-4, LLaMA-3, ResNet-50, ViT-L/14, BERT-large, Stable Diffusion,
Whisper, Segment Anything Model (SAM), AlphaFold2

**Named systems in papers:** Any capitalized acronym or coined name that the cited
paper defines as their contribution (e.g., "the MAMBA-based encoder they call SSML",
"the pipeline they term BioFuse")

### How to verify
The model or system name must appear in the extraction note, typically under a field
labeled "framework", "architecture", "model", "method", or "system name", or within
the findings text.

### Do NOT audit
- "a transformer-based encoder" (generic category, no proper noun)
- "a convolutional neural network" (generic)
- "deep learning" (too generic)
- "the authors' method" (no name given)

---

## 4. Sample / Item Count Claims

### What they are
A sentence that states a specific number of subjects, items, samples, or observations
associated with a named study or dataset.

### Examples across domains

**Clinical:** "a cohort of 1,524 patients", "830 whole-slide images from 3 centers"
**NLP:** "trained on 3.2 billion tokens", "annotated 12,000 sentence pairs"
**Computer vision:** "evaluated on 50,000 validation images", "a dataset of 2.3M frames"
**Survey research:** "N = 2,847 respondents across 12 countries"

### How to verify
The count must appear somewhere in the extraction note. Accept formats like "N=1524",
"1,524 patients", "approximately 1.5k subjects" (mark last as APPROX if original was
exact).

### Tolerance rules
- N < 100: match within ±5 (minor exclusions are common)
- N ≥ 100: exact match required (rounding at this scale implies a different number)
- Reported as "~X" or "approximately X": mark APPROX if within 5%

### Do NOT audit
- "a large cohort" (no number)
- "hundreds of participants" (no number)
- "thousands of training examples" (no number)

---

## 5. Study-Level Result Claims

### What they are
A sentence that credits a specific quantified finding to a named study (identified by
an adjacent citation key).

### Examples
- "Smith et al. showed a 12-point improvement over the baseline [@Smith2023]"
- "[@Jones2024] reported a 23% reduction in error rate when using multi-task learning"
- "the model reduced false positives by half compared to the prior state of the art [Brown2022]"

### How to verify
The citation key must be present immediately adjacent to the claim. Then check that
the effect magnitude (12 points, 23%, half) appears in the extraction note for that key.

### Common pitfall
A single sentence synthesizing results from multiple studies without per-study citation
keys — e.g., "several studies showed 10–20% gains [Smith2023; Jones2024; Brown2022]".
Each study's specific number needs its own extractable source. If the extraction only
says "improved accuracy", the specific percentage is NOT_IN_EXTRACTION.

---

## 6. Non-Auditable Sentence Types

Skip these — adding them to the audit table creates noise:

- **Methodological background**: "Attention mechanisms allow models to weight input
  tokens by relevance…" (textbook knowledge, no specific study claim)
- **Field-level generalizations**: "Large language models have transformed NLP…"
- **Transition sentences**: "The following section reviews multimodal fusion strategies…"
- **Qualitative limitation statements**: "A key limitation is the lack of prospective
  validation" (no number)
- **Future directions**: "Future work should explore…"
- **Synthesis without attribution**: "Across studies, the evidence suggests…" (no
  specific number and no citation key)
- **Author's own framing about the review**: "This review covers papers from 2015–2024…"

The heuristic: if removing the sentence would not change any specific factual claim
about a named study's results, it is not auditable.

---

## 7. Citation Key Formats

Different projects and drafting stages use different citation formats. The audit skill
should handle all of these:

| Format | Example | Notes |
|---|---|---|
| Author-year placeholder | `[Smith2023]` | Strip brackets to get filename stem |
| Pandoc citekey | `[@Smith2023]` | Strip `[@` and `]` |
| Multiple pandoc | `[@Smith2023; @Jones2024]` | Split on `;` and strip each |
| Numeric (Vancouver) | `[1]`, `[12,14]` | Requires a numbered reference list to resolve |
| Superscript numeric | `^1^`, `^12,14^` | Strip `^` and look up in reference list |
| Inline text | "Smith et al. (2023)" | Extract author + year directly |
| No citation | _(none)_ | Mark UNCITED — high severity |

When citation format is numeric, look for a references section at the end of the
manuscript, or a separate `references.md` / `.bib` file, to resolve number → author-year
→ extraction filename.

---

## 8. Searching Extraction Notes — Format-Agnostic Heuristics

Extraction notes may be strictly templated, semi-structured, or free-form prose.
Use these strategies regardless of format:

**For numbers:** Search for the exact number string (e.g., "0.87", "87%", "1,524").
Also try without comma separators ("1524") and with rounding variants ("0.9" for "0.87").

**For dataset names:** Search for the proper noun, case-insensitively. Also try
common acronym/expansion variants.

**For model names:** Search for the exact string, case-insensitively. Also try
removing punctuation (e.g., "GPT4" for "GPT-4").

**For sample counts:** Search for the number, then check surrounding text to confirm
it refers to sample size (not year, reference number, or something else).

**Structured fields to check first (if present):**
- Any line starting with `**` followed by a label ending in `:**`
- Section headers: `## Key Findings`, `## Results`, `## Methods`, `## Quantitative Evidence`,
  `## Performance`, `## Metrics`

**When a note is free-form prose:** Scan every line for numbers. For each number, note
the surrounding context. If the context matches what the manuscript claims, it is VERIFIED.

---

## 9. Severity Guide

Use this to prioritize what to report first:

| Status | Severity | Action needed |
|---|---|---|
| MISMATCH | Critical | Must correct before any further editing |
| NOT_IN_EXTRACTION | High | Either the note is incomplete or the claim was hallucinated |
| NO_EXTRACTION | High | Note may not have been created; claim is unverifiable |
| UNCITED | Medium | Add a citation or remove the specific claim |
| APPROX | Low | Standardize before final submission, not urgent |
| VERIFIED | None | No action needed |
