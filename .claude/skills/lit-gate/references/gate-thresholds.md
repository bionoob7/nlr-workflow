# Gate Thresholds Reference

Default values for each check, their severity levels, and how to override them.

---

## Default Threshold Table

| Check | Default Target | Severity | Notes |
|---|---|---|---|
| Word count (body-only) | ≤ 12,000 | CRITICAL | Body = everything except `## References` and below |
| Em-dash rate | ≤ 1.0 per 1,000 words | WARN | Equivalent to ≤ 10 em-dashes in a 10,000-word ms |
| Citation density | ≥ 10 per 1,000 words | WARN (< 10) / CRITICAL (< 5) | 1 citation per 100 words is baseline for reviews |
| Abstract present | YES | CRITICAL | Any section matching `/^#+ *abstract/i` |
| Major sections (##) | 4–12 | WARN | Below 4 = likely incomplete; above 12 = over-sectioned |
| Introduction subsections | 0 | WARN | Introductions should be flowing prose, not subsectioned |
| Figure callouts | ≥ 1 | WARN | Any "Figure N", "Fig. N", or "Fig N" in text |
| Table callouts | ≥ 1 | WARN | Any "Table N" or "Tab. N" in text |
| Unfilled placeholders | 0 | CRITICAL | `[PLACEHOLDER]`, `[INSERT]`, `[FIXME]`, `TODO`, `TBD` |
| Broken citation keys | 0 | CRITICAL | Empty `[@]` or unreplaced `[CITATION]`, `[REF]` |
| Citation format consistency | Consistent | WARN | Mixed pandoc + placeholder formats |
| NLR: paper-count claims (4a) | 0 | WARN | NLR projects only — skip for SLR projects |
| NLR: exhaustive assertions (4b) | 0 | WARN | NLR projects only — skip for SLR projects |
| NLR: meta-sentences (4c) | 0 | WARN | NLR projects only — skip for SLR projects |
| NLR: "corpus" term (4d) | ≤ 2 | WARN | NLR projects only — ≤ 2 avoids genre signal |

---

## Severity Levels

**CRITICAL** — A manuscript with this problem should not advance to the next step.
The issue is either a factual gap (no abstract, broken citations, unfilled text) or
a hard journal constraint violation (word count over limit). Fix and re-run the gate.

**WARN** — The manuscript can proceed, but the issue should be noted and ideally
resolved before final submission. Warnings often reflect advisory journal expectations
rather than hard requirements.

---

## Journal-Specific Overrides

Different journals have different hard limits. Common targets:

| Journal / Type | Word limit | Citation density | Em-dash guidance | Notes |
|---|---|---|---|---|
| npj Digital Medicine (Review) | 12,000 body | ≥ 10 / 1k words | ≤ 1 / 1k words | No abstract subsections |
| Nature Reviews (Review) | 8,000–10,000 | ≥ 12 / 1k words | ≤ 0.5 / 1k words | Strict; request limit increase if needed |
| Cell Reports Medicine | 6,000–8,000 | ≥ 8 / 1k words | ≤ 1 / 1k words | — |
| Short report / Letter | 2,000–3,000 | ≥ 8 / 1k words | ≤ 0.5 / 1k words | — |
| Thesis chapter | 8,000–15,000 | ≥ 6 / 1k words | Flexible | — |
| Conference paper | 6,000–10,000 | ≥ 8 / 1k words | Flexible | — |

Use these as starting points; always verify against the journal's current author guidelines.

---

## How to Override Thresholds

### Option 1 — Inline at invocation
The user can name targets when calling the skill:

```
/lit-gate 10000              # sets word count target to 10,000
/lit-gate 10000 citations:15 # sets word count to 10,000 and citation density to ≥15/1kw
```

Extract these from the user's message before running measurements.

### Option 2 — Project config file
Create a file named `gate_config.md` in the project root or manuscript directory.
The skill reads this file in Step 1. Format:

```markdown
# Gate Configuration

word_target: 10000
emdash_target: 0.5
citation_density_target: 12
abstract_required: true
figure_callouts_required: true
table_callouts_required: true
```

Any key not present uses the default value.

### Option 3 — Verbal override during the run
If the user says "our journal allows 15,000 words" or similar during a conversation,
update the target for that check before measuring and note the override in the report.

---

## Computed Thresholds

Some thresholds scale with manuscript length:

**Em-dash target (absolute)**
```
emdash_absolute_target = (body_word_count / 1000) × emdash_rate_target
# Default: (body_word_count / 1000) × 1.0
# Example: 10,000-word ms → ≤ 10 em-dashes absolute
```

**Citation target (absolute)**
```
citation_absolute_target = (body_word_count / 1000) × citation_density_target
# Default: (body_word_count / 1000) × 10
# Example: 10,000-word ms → ≥ 100 total citations
```

Always report both the rate (per 1,000 words) and the absolute count in the scorecard,
since journal instructions sometimes specify one or the other.

---

## Measurement Notes

### Word count
The `wc -w` command counts whitespace-separated tokens. This slightly overcounts
(hyphenated phrases count as multiple words) and undercounts (words inside image alt
text or HTML tags). For most manuscripts the difference is ≤ 2%. Use body-only count
(excluding `## References` / `## Bibliography` and below) as the primary number.

To exclude the references section from the word count:
```bash
# Stop counting at the References section
sed -n '1,/^## References\|^## Bibliography\|^# References/p' manuscript.md | \
  grep -v "^#" | grep -v "^[[:space:]]*$" | wc -w
```

### Citation counting
The Grep patterns for citations work as follows:
- Pandoc format `[@key]`: matches `\[@[^]]*\]` — finds all citation calls including
  multi-cite `[@key1; @key2]`
- Placeholder format `[Smith2023]`: matches `\[[A-Z][a-zA-Z]+[0-9]{4}[a-z]?\]`

Count of citation *calls* (how many times citations appear in text) is what matters
for density — not the number of unique papers cited.

### Em-dash detection
The em-dash character is `—` (U+2014). En-dash `–` (U+2013) and hyphen `-` (U+002D)
are not counted. Some manuscripts use `--` for em-dash; check both:

```bash
grep -o "—\|--" manuscript.md | wc -l
```

If the manuscript uses `--` consistently, count those too and note it in the report.

---

## Pass/Fail Decision Matrix

```
Are any CRITICAL checks failed?
  YES → VERDICT: FAIL
        List all critical failures with specific corrective actions
        Instruct user to re-run /lit-gate after fixing

  NO → Are any WARN checks failed?
        YES → VERDICT: WARN
              List all warnings with advisory notes
              Confirm user can proceed to next step

        NO → VERDICT: PASS
             Confirm ready to proceed
```

---

## NLR Language Check — Detection and Thresholds

This check applies **only to narrative literature reviews**. Check CLAUDE.md before
running. If CLAUDE.md says "narrative review" (or "NLR"), run the check. If it says
"systematic review", "scoping review", or is silent on the distinction, skip it.

### Why this check exists

A narrative review synthesizes arguments; a systematic review enumerates evidence.
When the two genres are mixed, editors recognize the mismatch from specific prose
patterns: exact counts of the review's paper collection ("89 studies in this corpus"),
exhaustive-coverage assertions ("none of the 280 reviewed studies reports…"), and
meta-sentences that describe the paper set as a numbered dataset ("this review
synthesizes 280 studies published between 2019 and 2026"). These patterns are
acceptable in an SLR methods section; they are a genre error in a narrative review.

### Sub-pattern targets and detection

| Sub-pattern | Label | Grep command | Target | Severity |
|---|---|---|---|---|
| Paper-count claims | 4a | `grep -icE "[0-9]+ (papers?\|studies\|articles?) in (this\|the) corpus"` | 0 | WARN |
| | | `grep -icE "only [0-9]+ (papers?\|studies) (attempt\|achieve\|integrate\|report\|address)"` | 0 | WARN |
| Exhaustive assertions | 4b | `grep -icE "(none\|all) of the [0-9]+ reviewed (papers?\|studies)"` | 0 | WARN |
| | | `grep -icE "absent from all [0-9]+ (papers?\|studies)"` | 0 | WARN |
| Meta-sentences | 4c | `grep -icE "this review (synthesizes\|examines\|covers\|presents\|includes) [0-9]+"` | 0 | WARN |
| "corpus" term | 4d | `grep -ic "in this corpus\|in the corpus\|the corpus"` | ≤ 2 | WARN if > 2 |

### Action when targets are exceeded

List every matching line in the gate action list. Direct the user to run
`/lit-deregister` — that skill's Category 4 contains the rewrite strategies.

Do not attempt to fix the language during a gate run. The gate is a measurement
checkpoint; rewriting belongs to the deregister pass.

### What NOT to flag

Numbers from *cited studies* — patient counts, AUC values, cohort sizes — are never
SLR language and must not be flagged:
- ✓ "validated on 2,279 patients across 12 centers [Huang2025]" — citable empirical claim
- ✗ "89 studies in this corpus integrate pathology with genomics" — review corpus count

Numbers that enumerate the review's own *arguments* (not papers) are also not SLR
language:
- ✓ "Three barriers limit clinical translation" — argument enumeration, keep as-is
