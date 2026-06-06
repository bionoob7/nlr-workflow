---
name: lit-deregister
description: >
  Removes AI writing fingerprints from academic manuscript drafts. Scans the text
  for four categories of AI register problems — antithesis constructions ("not X
  but Y"), editorializing adjectives ("striking", "remarkable", "unprecedented", etc.),
  repetitive structural refrains, and systematic-review language in a narrative review
  (paper counts like "89 studies in this corpus", exhaustive-corpus assertions like
  "none of the 280 reviewed studies", and meta-sentences like "this review synthesizes
  280 studies") — then rewrites only the excess instances to bring counts within target
  thresholds. Produces the revised manuscript plus a before/after change table. Use
  this skill after factual verification and before any final style polish, or any time
  a manuscript reads as AI-generated or as a systematic review when it should be a
  narrative review. Works on any markdown manuscript regardless of domain.
  Invoke with /lit-deregister.
allowed-tools: Read Glob Grep Write Bash
metadata:
  version: "1.0"
---

# Lit-Deregister — AI Register Fingerprint Remover

Academic prose written or heavily assisted by AI tends to accumulate four types of
register problems that human reviewers and journal editors can reliably spot:

1. **Antithesis constructions** — "not X, but Y" sentences that create rhetorical
   drama where plain statement would do.
2. **Editorializing adjectives** — intensifiers like "striking", "remarkable", and
   "unprecedented" that assert significance rather than demonstrate it.
3. **Repetitive structural refrains** — phrases or sentence-opening patterns that
   appear so frequently they read like a tic.
4. **Systematic review language in a narrative review** — paper counts describing
   the review's own corpus ("89 studies in this corpus"), exhaustive-corpus assertions
   ("none of the 280 reviewed studies"), and meta-sentences about the review's scope
   ("this review synthesizes 280 studies published between…"). These patterns signal
   a genre mismatch: a systematic review posture inside a narrative review article.

Your job is to find these, count them, rewrite the excess, and save a clean manuscript
with a record of what changed. You are editing prose, not facts — never change a
number, citation, or empirical claim during this pass.

Read `references/ai-patterns.md` before starting. It contains the full pattern list,
count targets, detection strategies, and rewrite guidance.

---

## Step 1 — Locate the Manuscript

Check these locations in order, stopping at the first match:

1. A path the user explicitly named
2. `manuscript/manuscript_v1.md`
3. `manuscript/manuscript_cited.md`
4. Any `manuscript/*.md` file
5. Any `.md` file in the current directory with > 3,000 words and section headers

If multiple candidates exist, list them and ask the user which to process.

Record: `source_path` (input), `output_path` (where to save the revised manuscript).
Save the revised file with a version suffix: if input is `manuscript_v1.md`, output
is `manuscript_v2.md`; if input is `manuscript_cited.md`, output is
`manuscript_cited_deregistered.md`. Do not overwrite the original.

---

## Step 2 — Read the Pattern Definitions

Read `references/ai-patterns.md`. It defines:
- The exact patterns to search for in each category
- Count targets (how many instances are acceptable)
- Rewrite strategies for each pattern type
- Patterns that are never acceptable (target = 0)

Keep this reference open — you will need it throughout Steps 3–5.

---

## Step 3 — Count-and-Flag Pass (Read-Only)

Before rewriting anything, do a complete read-only scan to count all instances of
every pattern. This gives you a baseline so you know the scale of the problem and
can prioritize where to focus rewrites.

For each pattern category:

**3a. Antithesis constructions**
Use Grep or read the manuscript and search for these patterns (case-insensitive):
- `not .{1,60} but ` (the core "not ... but" structure)
- `not .{1,40}; (rather|instead|it is|they are)`
- `not merely .{1,40} but`
- `not simply .{1,40} but`
Record every matching sentence and its section.

**3b. Editorializing adjectives**
Search for the full list from `references/ai-patterns.md`. For each word, count
total occurrences across the manuscript. Record location (section + surrounding clause).

**3c. Repetitive refrains**
Scan for any phrase of 3+ consecutive words that appears in 4 or more different
paragraphs. Also scan for sentence-opening patterns (first 5 words of each sentence)
that repeat 4+ times. Record the phrase and all locations.

**3d. Systematic review language** (run only if the project is a narrative review —
check CLAUDE.md or ask the user if unsure)

Use Grep or read the manuscript and count instances of each sub-pattern:

```bash
# 4a: exact paper-count claims about the corpus
grep -icE "[0-9]+ (papers?|studies|articles?) in (this|the) corpus" [manuscript_path]
grep -icE "the [0-9]+ (papers?|studies)" [manuscript_path]
grep -icE "only [0-9]+ (papers?|studies) (attempt|achieve|integrate|report|address)" [manuscript_path]

# 4b: exhaustive-corpus assertions
grep -icE "(none|all) of the [0-9]+ reviewed (papers?|studies)" [manuscript_path]
grep -icE "absent from all [0-9]+ (papers?|studies)" [manuscript_path]

# 4c: meta-sentences about the review's scope
grep -icE "this review (synthesizes|examines|covers|presents|includes) [0-9]+" [manuscript_path]

# 4d: "corpus" as a formal SLR boundary term
grep -ic "in this corpus\|in the corpus\|the corpus" [manuscript_path]
```

Record the total count for each sub-pattern and all matching sentences.

After the scan, build a **count table**:

| Category | Pattern | Count | Target | Over by |
|---|---|---|---|---|
| Antithesis | "not X but Y" variants | 14 | ≤ 2 | 12 |
| Editorializing | "striking" | 7 | 0 | 7 |
| Editorializing | "remarkable" | 4 | 0 | 4 |
| Refrain | "As demonstrated above, …" opening | 5 | ≤ 2 | 3 |
| SLR language | paper-count claims (4a) | 6 | 0 | 6 |
| SLR language | exhaustive-corpus assertions (4b) | 3 | 0 | 3 |
| SLR language | meta-sentences (4c) | 1 | 0 | 1 |
| SLR language | "in this/the corpus" (4d) | 18 | ≤ 2 | 16 |

Print this table to the user immediately. Ask: "These are the patterns I found.
Shall I proceed with rewrites, or do you want to adjust any targets first?"

Wait for confirmation before proceeding.

---

## Step 4 — Rewrite the Excess

Work through the manuscript once, rewriting instances in excess of their targets.
Apply rewrites in this priority order:

1. **Editorializing adjectives** (target = 0 for the items on the prohibited list;
   other intensifiers target ≤ 3 per manuscript)
2. **Antithesis constructions** (target = ≤ 2 per manuscript)
3. **Repetitive refrains** (target = ≤ 3 occurrences for any single phrase)
4. **Systematic review language** (target = 0 for paper-count claims, exhaustive
   assertions, and meta-sentences; ≤ 2 for "in this/the corpus" — only if this is
   a narrative review project; skip this category for SLR projects)

For each rewrite, follow the guidance in `references/ai-patterns.md`. The general
principles are:

- **Antithesis**: Replace the "not X, but Y" structure with a direct positive
  statement of Y. The contrast is implicit once Y is stated clearly.
- **Editorializing adjective**: Delete the adjective and let the evidence speak.
  If the sentence feels thin without it, the underlying claim needs strengthening
  with a specific result, not a stronger adjective.
- **Refrain**: Rephrase the repeated sentence opening using a different grammatical
  structure. Vary: subordinate clause → participial phrase → direct statement.
- **SLR language**: Replace paper counts with qualitative scope descriptors
  ("the dominant evidence base", "relatively few published studies"); replace
  exhaustive-corpus assertions with field-level gap language ("the reviewed literature
  contains no…", "no published study documents…"); remove or integrate meta-sentences
  about the review's scope. See Category 4 in `references/ai-patterns.md` for
  rewrite examples.

**What not to change:**
- Numbers, percentages, or metric values *from cited studies* (patient counts,
  AUC values, cohort sizes — these are auditable empirical claims, not SLR language)
- Citation keys or reference placeholders
- Quoted text from other papers
- Technical terminology (even if it sounds dramatic in context, e.g., "critical
  pathway" in a biological sense)
- Section headings

**Judgment on antithesis:** Keep the 2 best-placed antithesis constructions — the
ones where the contrast is genuinely the most direct way to convey the distinction.
Rewrite the rest.

**Judgment on SLR language:** Numbers in the Abstract that summarize the review's
three-tier argument structure (e.g., "Three barriers limit clinical translation")
enumerate arguments, not papers — keep these. Numbers that describe the review's
own paper collection are SLR language — rewrite.

---

## Step 5 — Save Outputs

**5a. Save the revised manuscript** to `output_path` (determined in Step 1).

**5b. Save a change report** to the same directory as the output manuscript:
`deregister_report_YYYYMMDD.md`

Use this report structure:

```markdown
# De-Register Pass Report

**Input manuscript:** [source_path]
**Output manuscript:** [output_path]
**Date:** YYYY-MM-DD

---

## Count Summary

| Category | Pattern | Before | After | Target |
|---|---|---|---|---|
| Antithesis | all "not X but Y" variants | 14 | 2 | ≤ 2 |
| Editorializing | "striking" | 7 | 0 | 0 |
| Editorializing | "remarkable" | 4 | 0 | 0 |
| Refrain | "As demonstrated above, …" | 5 | 2 | ≤ 3 |
| SLR language | paper-count claims (4a) | 6 | 0 | 0 |
| SLR language | exhaustive-corpus assertions (4b) | 3 | 0 | 0 |
| SLR language | meta-sentences (4c) | 1 | 0 | 0 |
| SLR language | "in this/the corpus" (4d) | 18 | 2 | ≤ 2 |

---

## Changes Made

### Antithesis rewrites

**Change 1 of N**
Section: [section heading]
Before: "[original sentence]"
After:  "[revised sentence]"

[repeat for each change]

### Editorializing adjective removals

**Change N of N**
Section: [section heading]
Before: "[original clause]"
After:  "[revised clause]"

### Refrain rewrites

[same format]

### Systematic review language rewrites

**Change N of N**
Section: [section heading]
Pattern type: [4a paper-count / 4b exhaustive-corpus / 4c meta-sentence / 4d corpus term]
Before: "[original sentence]"
After:  "[revised sentence]"

[repeat for each change]

---

## Patterns Kept (Within Target)

List the antithesis constructions and other patterns that were retained, with brief
notes on why they were kept:

- [sentence] — kept because [reason: e.g., "contrast is the most direct framing",
  "term has a specific technical meaning here", "number belongs to a cited study,
  not to the review's paper collection"]

---

## Patterns Approaching Target (Monitor)

List any pattern that is at the target limit (not over, but worth watching if the
manuscript is revised again):
```

---

## Step 6 — Report to User

After saving, print:

```
De-register pass complete.

Revised manuscript: [output_path]
Change report:      [report_path]

Changes made:
  Antithesis rewrites:          N  (N remaining, target ≤ 2)
  Editorializing adj. removed:  N  (0 remaining)
  Refrain rewrites:             N  (N remaining for worst phrase)
  SLR language rewrites:        N  (paper-count: N remaining / exhaustive: N remaining /
                                    meta-sentence: N remaining / corpus-term: N remaining)

Review the change report before proceeding — verify that no rewrite altered an
empirical claim or changed a sentence's meaning. In particular: numbers from cited
studies (patient counts, AUC values) must not have been touched; only numbers
describing the review's own paper collection should have been rewritten.
```

---

## Constraints

- **Never change facts.** If a rewrite would require altering a number, citation,
  or specific empirical claim, rephrase around the fact rather than changing it.
- **Preserve the author's argument.** The goal is to remove register problems, not
  to rewrite for style. When in doubt, make the minimum change.
- **One pass only.** Do not re-scan the output for new problems introduced by
  rewrites — that belongs in a separate pass.
- **When a pattern is genuinely the best phrasing**, keep it and note it in the
  "Patterns Kept" section of the report. The targets are thresholds, not mandates.
