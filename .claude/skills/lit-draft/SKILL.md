---
name: lit-draft
description: >
  Drafts a single named section of an academic literature review manuscript, guided
  by the word budget in the project outline, content from per-paper extraction notes,
  and strict quality constraints (citation density, numeric accuracy, register,
  em-dash policy). Use this whenever any section of a review manuscript needs to be
  written or re-written — introduction, body section, or conclusion. Invoke by
  naming the section: /lit-draft 3 (section number), /lit-draft 3.1 (subsection),
  or /lit-draft "Methods Integration" (section title). Automatically locates the
  project outline, reads relevant extraction files, verifies every cited number
  against the source notes, runs a register self-check, and saves the draft to the
  draft/ directory. Works for any literature review project that keeps a section
  outline and per-paper extraction notes, regardless of topic or journal target.
  Invoke with /lit-draft [section identifier].
allowed-tools: Read Glob Grep Write Bash
metadata:
  version: "1.1"
---

# Lit-Draft — Section Drafter for Literature Reviews

This skill drafts one section of a literature review at a time. The quality controls
are enforced inside the drafting loop, not as post-hoc recommendations: every number
is verified against an extraction file before it enters the prose, the register is
checked before saving, and the word count is verified against the outline budget.

The most important constraint is **numeric accuracy**. A wrong AUC, an invented
patient count, or a misattributed model name is worse than a qualitative claim — it
introduces scientific error that is hard to catch in review. This skill trades speed
for verifiability by reading extraction files explicitly rather than generating
numbers from memory.

Read `references/quality-constraints.md` for exact threshold values and worked
examples of constraint violations and their rewrites.

---

## Step 1 — Locate the Project Outline

The outline tells you the section's word budget, argument description, and key papers.
Check in this order:

1. Path the user explicitly named in their message
2. `outline/outline_current.md`
3. `outline/outline_v*.md` (most recently modified)
4. `outline.md` or `OUTLINE.md` in the project root
5. Any `*.md` file in an `outline/` directory

If multiple candidates exist and none is named `outline_current.md`, ask the user
which one to use.

Also locate the project style guide (usually `CLAUDE.md` or `STYLE.md` in the
project root). Read it for voice, citation format, and any project-specific rules.
If no style guide exists, apply the defaults in Step 5.

---

## Step 2 — Parse the Section Identifier

The user invokes the skill with a section identifier. Match it against the section
headings in the outline. It may be:

- A section number: `3` or `3.1`
- A quoted or unquoted title keyword: `"Pathomic Fusion"`, `radiogenomics`, `conclusion`
- A short slug: `intro`, `methods`, `discussion`

From the matched section, extract:
- **Full section title** (as it appears in the outline)
- **Target word count** — look for phrases like "Target word count: N", "~N words",
  "N words", or "budget: N" in the section's outline entry
- **Subsections** — any `###`-level headings within the section entry
- **What it covers** — the description paragraph in the outline entry
- **Key papers** — any paper list in the section entry (often labeled "Key papers:",
  "Papers anchoring this section:", "Feeds from:", etc.)
- **Figure references** — whether any "(Figure X)" callout belongs in this section
- **Next section title** — the section that immediately follows (for the closing
  transition sentence)

If no word budget is found in the outline entry, use these defaults based on section
type: Introduction 900–1,200; conclusion 600–900; short body section 600–900;
standard body section 1,000–1,500; long body section 1,500–2,500.

If the section has subsections and its total budget exceeds 1,800 words, recommend
drafting subsection by subsection and ask the user to confirm which subsection to
start with.

State the resolved fields before proceeding:
```
Section:     [full title]
Budget:      [N] words (±10%)
Subsections: [list, or "none"]
Output:      draft/[section_slug].md
```

---

## Step 3 — Discover and Read Extraction Files

Per-paper extraction notes are the source of all verifiable facts used in the draft.
Locate the extraction directory by checking these names in the project root:
`extractions/`, `notes/`, `papers/`, `summaries/`, `readings/`.

Use the first one that exists and contains `.md` files.

Read one sample extraction file to discover the format used in this project. Common
formats include:
- Structured templates with `## Metadata`, `## Methods`, `## Key Findings` headers
- Flat notes with `Title:`, `Authors:`, `Key metric:` labels
- Free-form bullet notes

From the sample, identify which fields carry: author/year, metric values, dataset
names, patient counts, architecture/model names. Use these field names when reading
subsequent extraction files.

Then, for each paper in the outline's key papers list:
1. Glob for `[extraction_dir]/[AuthorYear]*.md` (try case variants)
2. Read the file if found; record it as "has extraction"
3. If not found, note as "no extraction — qualitative only"

Build an internal lookup:
```
For each paper with an extraction file:
  - architecture_name:  exact string from the architecture/framework field
  - dataset_names:      exact strings from the datasets field
  - patient_counts:     exact N values from metrics or dataset detail fields
  - metric_values:      exact {metric, value, dataset, N} tuples from the metrics field
  - key_claim:          verbatim claim recommended for citation
```

This lookup is the **only** permitted source for specific numbers in the draft.

---

## Step 4 — Plan the Argument Structure

Before writing prose, sketch an argument plan (internal scaffolding; it will not
appear in the draft).

For each paragraph you intend to write:
- What claim does this paragraph advance?
- Which papers provide evidence for this claim?
- What specific verified facts from the lookup will you include?
- What is the logical connector to the next paragraph?

This step exists to catch two failure modes before they reach the prose:
(a) a paragraph that lists papers without advancing a claim — the "annotated
bibliography" failure mode — and (b) a paragraph that makes a claim without any
verifiable evidence from the extraction lookup.

For sections with subsections: plan all subsections before writing any of them.
This ensures the section reads as a unified argument rather than a sequence of
independent mini-reviews.

---

## Step 5 — Write the Draft

Apply these constraints while drafting. Each has a reason explained below — the
reason helps you handle cases not covered by the rule literally.

### 5a — Word Count

Stay within the outline budget ±10%. The budget exists because the whole manuscript
has a target word count, and each section's budget is allocated to fit within it.
An over-budget section forces cuts elsewhere and risks disrupting the manuscript's
overall balance.

After drafting, verify the count:
```bash
wc -w draft/[section_slug].md
```

### 5b — Narrative Synthesis Style

Papers serve arguments; arguments do not serve papers. Every paragraph leads with
a claim, deploys papers as evidence, then interprets what the evidence means for the
review's thesis. The prohibited pattern narrates papers one by one without a leading
claim:

  ✗ "Smith et al. (2023) found X. Jones et al. (2024) found Y. Wang (2024) found Z."

The preferred pattern states the argument first:

  ✓ "Multimodal survival models consistently outperform their unimodal counterparts
    on independent validation cohorts [Smith2023; Jones2024]. The magnitude of
    this advantage varies by endpoint and cancer type: for overall survival,
    Wang et al. report a C-index of [verified value] on [verified dataset] [Wang2024],
    while..."

### 5c — Citation Density and Format

Every factual claim about a specific study requires an immediate citation. Target
at least 1 citation per 60–80 words of body text in technical sections. The reason:
a reader checking a claim — or a later audit pass — needs to locate the source
without hunting through the full paper list.

Citation placeholder format: `[FirstAuthorYear]` (e.g., `[Chen2023]`).
Multiple citations in one bracket: `[Smith2023; Jones2024]`.

**Citation placement rule:** Citation placeholders will become numbered superscripts
or parenthetical markers when typeset. A sentence must never *open* with a placeholder,
because that produces a citation marker as the first character of a sentence.

Two correct patterns:

- **Author named in prose:** Write the author name out; place the placeholder at the
  end of the relevant clause or sentence.
  ✓ "Chen et al. achieved a C-index of [value] on [dataset] ([N] patients) [Chen2023]."
  ✓ "Wang et al. report that integrating radiology with pathology improves C-index
    by [value] over the unimodal baseline [Wang2024]."

- **Author not named:** Cite inline after the claim, before the period or comma.
  ✓ "Integrating radiology with pathology improves C-index over the unimodal baseline
    [Wang2024]."
  ✓ "Three studies confirm this pattern [Smith2023; Jones2024; Wang2024]."

  ✗ "[Chen2023] achieved a C-index of 0.74..." — placeholder as sentence opener
  ✗ "[Wang2024] report that..." — placeholder as subject of a verb

For each cited study in a technical sentence, include the specific metric, dataset,
and patient count **if they are in the lookup**:

  ✓ "Chen et al. achieved a C-index of [value from lookup] on [dataset from lookup]
    ([N from lookup] patients) [Chen2023]."
  ✗ "Chen et al. showed strong performance [Chen2023]." — too vague in a technical section

### 5d — Numeric Accuracy

Every specific number — metric value, patient count, dataset name, model/architecture
name — must be traceable to the lookup built in Step 3. Before writing a number:
locate it in the lookup. If it is absent, write the qualitative claim instead.

  ✓ "Chen et al. achieved a C-index of 0.74 on TCGA-BLCA (N=412) [Chen2023]."
    — only if 0.74 and 412 are confirmed in the extraction note
  ✓ "Chen et al. demonstrated improved survival discrimination on TCGA-BLCA
    relative to the unimodal baseline [Chen2023]." — always safe if the direction
    is confirmed in Key Findings, even when no metric value is in the extraction note
  ✗ "Chen et al. achieved a C-index of 0.74 [Chen2023]." — if 0.74 comes from
    model memory rather than the extraction lookup

Inventing plausible-sounding numbers is the highest-severity error in this workflow.
A wrong metric, attributed to the wrong paper, is worse than no number at all.

### 5e — Em-Dash Policy

Limit em-dashes (—) to ≤1 per 1,000 words of the section. Em-dashes are reserved
for a sharp interruption or appositive where a comma or parentheses would genuinely
weaken the sentence. Do not use them to attach subordinate clauses, introduce
examples, or replace colons.

When tempted to write an em-dash, try the alternative first: comma, parentheses,
colon, or a new sentence. If the alternative reads just as well, use it.

### 5f — No Bullet Lists in Main Text

Main text is prose only. No bullet points, no numbered lists, no dashes as list
markers. If you find yourself wanting to list three items, integrate them into a
sentence with "first... second... third..." or with a colon and a compound structure.

### 5g — Section-Type Rules

**Introduction:** No subsection headings. Flowing paragraphs only. By the end of the
second paragraph, at least one named study, dataset, or method with a specific metric
should appear — the introduction must not be purely scene-setting. Avoid opening
with "Recently..." or "In recent years..." or any sentence that contains no named entity.

**Body sections with subsections:** Each subsection opens with a heading followed
immediately by the argument the subsection advances. The first sentence states the
claim; the rest of the subsection marshals evidence.

**Conclusion:** No subsection headings. Does not repeat what was said in body sections.
Synthesizes the pattern across papers — what the accumulation of evidence reveals
together — and ends with a concrete forward-looking statement naming a specific
technology, mechanism, or design choice that will unlock the next step.

### 5h — Figure Call-Outs

Where the outline specifies that a figure belongs in this section, insert a
parenthetical call-out at the logical point in the text:

  "...the three integration strategies are shown schematically (Figure 2)."

The call-out should appear where the reader would benefit from the figure, not at an
arbitrary sentence boundary.

### 5i — Closing Transition

End the section (or subsection) with 2–3 sentences that transition to the next
section. A good transition briefly characterizes what was established, then names
the specific gap or question that motivates the next section. Generic transitions
should be replaced with argument-advancing ones:

  ✗ "The next section examines radiogenomics approaches."
  ✓ "The pathology-centric literature has largely treated radiology as a secondary
    validation layer; the radiogenomics literature inverts this priority and asks
    what imaging can reveal about molecular state without tissue biopsy."

### 5j — Narrative Review Language (NLR projects only)

If CLAUDE.md identifies this project as a narrative review, do not write in the
voice of a systematic review. The distinction matters because narrative reviews
synthesize arguments, whereas systematic reviews enumerate evidence — and editors
can tell the difference from the prose register alone.

**Paper counts about the review corpus** — do not write:
  ✗ "89 studies in this cluster integrate pathology with genomics"
  ✗ "only 12 of the 280 reviewed papers attempt three-modality fusion"
Write qualitative scope descriptors instead:
  ✓ "the dominant evidence base integrates pathology with genomics"
  ✓ "relatively few published studies attempt three-modality fusion"

**Exhaustive-corpus assertions** — do not write:
  ✗ "none of the 280 reviewed studies reports a prospective result"
  ✗ "prospective validation is absent from all papers in the corpus"
Use field-level gap language instead:
  ✓ "the reviewed literature contains no prospective results"
  ✓ "prospective validation has not been reported"

**Meta-sentences about the review's paper collection** — do not open with:
  ✗ "This section reviews 45 radiogenomics papers published between 2020 and 2025."
Describe the literature's scope qualitatively:
  ✓ "Radiogenomics has evolved from single-institution correlation studies to
    multi-cohort validation frameworks."

**"corpus" as an SLR boundary term** — do not write "in this corpus" or "the corpus"
as a formal label for the review's paper set. Write "in the reviewed literature",
"in published work", or "among the studies examined here."

Numbers from *cited studies* are always permitted — patient counts, AUC values,
cohort sizes attributed to a specific paper are auditable empirical claims, not SLR
language. The prohibition applies only to numbers that describe the *review's own
paper collection.* See `references/quality-constraints.md` §8 for worked examples.

---

## Step 6 — Register Self-Check

Before saving, count and fix two specific failure patterns. These patterns are
common in AI-assisted academic writing and signal machine-generated text to editors.

**Check 1 — Antithesis constructions (max 1 per section)**

Pattern: sentences containing `not [X] but [Y]`, `not [X]; it is [Y]`, `rather than
[X], [Y]` as a rhetorical emphasis device.

```bash
grep -iE "\bnot\b.{5,80}\bbut\b" draft/[slug].md | wc -l
```

If the count exceeds 1, rewrite the extras as plain declaratives. State Y directly;
if the contrast matters, place it in a subordinate clause rather than as the main
frame.

**Check 2 — Editorializing adjectives on findings (target 0 prohibited)**

Prohibited (0 instances): striking, remarkable, unprecedented, groundbreaking,
particularly consequential, not a minor caveat, precisely the, crucially (as sentence
opener), compellingly, transformative (applied to a finding or result).

For each instance, delete the editorial frame. The specific fact should carry the
weight:

  ✗ "The striking finding that AUC reached 0.94..."
  ✓ "The model achieved AUC=0.94..."

**Check 3 — Systematic review language (NLR projects; target 0)**

If the project is a narrative review, grep for SLR-genre patterns before saving:

```bash
grep -icE "[0-9]+ (papers?|studies|articles?) in (this|the) corpus" draft/[slug].md
grep -icE "(none|all) of the [0-9]+ reviewed (papers?|studies)" draft/[slug].md
grep -icE "this (section|review) (synthesizes|examines|covers|includes) [0-9]+" draft/[slug].md
grep -ic "in this corpus\|in the corpus" draft/[slug].md
```

Target: 0 for the first three patterns; ≤ 2 for the "corpus" term. Any match above
target must be rewritten before saving — apply constraint 5j rewrites.

---

## Step 7 — Save the Draft

### Deriving the section slug

Generate the slug from the section title using this algorithm:
1. Extract the section number if present (e.g., "3.1" → `03_1`)
2. Take the first 3–4 meaningful words of the title; drop articles, prepositions,
   conjunctions
3. Lowercase; replace spaces and punctuation with underscores
4. Combine: `[zero-padded section number]_[short title].md`

Examples:
- "1. Introduction" → `01_introduction.md`
- "3. Pathology-Centric Multimodal Integration" → `03_pathology_centric.md`
- "3.1 — Pathomic Fusion: Histology Meets Genomics" → `03_1_pathomic_fusion.md`
- "Challenges, Gaps, and Future Directions" (§6) → `06_challenges.md`
- "Methods" (no number given) → `methods.md`

Save to `draft/[slug].md`. If the `draft/` directory does not exist, create it.

### Draft metadata footer

Append a metadata comment block at the end of the file. This block is not part of
the manuscript and should be stripped during the assembly step:

```markdown
---
<!-- DRAFT METADATA — remove before manuscript assembly -->
Section:              [full section title]
Word count:           [N] (budget: [budget] ±10% → max [budget×1.1])
Em-dash count:        [N] (limit ≤[ceil(budget/1000)])
Register check:       antithesis=[N], editorializing=[N], SLR-language=[N]
Papers cited:         [N unique [FirstAuthorYear] placeholders]
No-extraction papers: [list papers cited as qualitative-only, or "none"]
-->
```

---

## Step 8 — Report to User

After saving, print:

```
Draft saved: draft/[slug].md

Section:        [title]
Words:          [N] / [budget] (max [budget×1.1])    [PASS / OVER BUDGET]
Em-dashes:      [N] / ≤[limit]                       [PASS / OVER LIMIT]
Antithesis:     [N] / ≤1                             [PASS / REWRITE NEEDED]
Editorializing: [N] / 0 prohibited                   [PASS / REWRITE NEEDED]
SLR language:   [N] / 0 (corpus-term ≤2)             [PASS / REWRITE NEEDED]

Papers cited:   [N]
Missing extractions (qualitative only): [list or "none"]

Next section:   [title of next section in outline]
Run: /lit-draft [next section identifier]
```

---

## Constraints

- **Never invent numbers.** If a value is not in the extraction lookup, write the
  qualitative claim. A wrong number is worse than no number.
- **Never use bullet lists in main text.** Prose only — including for findings,
  limitations, or comparison items.
- **Read extraction files before drafting.** Do not rely on memory of paper content
  from earlier in the conversation. Re-read the relevant files each invocation,
  because the lookup needs to be current and complete.
- **One section per invocation.** Do not draft multiple sections in one run. The
  quality checks require focused attention; batching sections degrades both accuracy
  and prose coherence.
- **Do not modify the outline.** If you discover a gap or inconsistency in the
  outline while drafting, report it to the user rather than silently fixing it.
- **Do not assume a specific extraction format.** Discover the format by reading a
  sample file in Step 3. Different projects use different templates.
