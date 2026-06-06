---
name: lit-gate
description: >
  Quantitative submission-readiness gate for academic manuscript drafts. Runs a
  battery of objective, tool-based checks — word count, em-dash frequency, citation
  density, abstract presence, section structure, figure callouts, unfilled placeholders,
  broken citation keys, and (for narrative reviews) systematic-review language patterns
  — then produces a structured pass/fail scorecard and a verdict. Unlike asking Claude
  to "check the manuscript", this skill uses Bash and Grep to measure the actual file,
  so the results are verifiable and not self-reported. Use before any submission-prep
  step, after all editing passes are complete, or any time you need a quick health
  check on a draft. Works on any markdown manuscript.
  Invoke with /lit-gate, or /lit-gate [word-limit] to set a custom word count target.
allowed-tools: Read Glob Grep Write Bash
metadata:
  version: "1.0"
---

# Lit-Gate — Manuscript Submission-Readiness Checker

The purpose of this skill is to replace "the AI says it looks good" with "the file
actually measures within target." Every check here is run with a tool, not eyeballed.

The gate has two verdict levels:
- **PASS** — All critical checks are within target. Ready to proceed.
- **FAIL** — One or more critical checks failed. Do not proceed until fixed.
- **WARN** — All critical checks pass, but one or more advisory checks are out of range.
  User decides whether to proceed.

Read `references/gate-thresholds.md` for the full default threshold table and how
to customize them. The thresholds listed in Step 2 below are the defaults; override
them if the user specified different targets or if a project-level config file exists.

---

## Step 1 — Locate the Manuscript

Check these locations in order, stopping at the first match:

1. A path the user explicitly named in their message
2. `manuscript/manuscript_v2.md` (post-deregister version, if exists)
3. `manuscript/manuscript_v1.md`
4. `manuscript/manuscript_cited.md`
5. Any `manuscript/*.md` file (if multiple, list them and ask)
6. Any `.md` file in the current directory with > 3,000 words and `## ` section headers

Also check whether the user passed a word count target inline, e.g., `/lit-gate 10000`
or "run lit-gate with a 10,000 word limit". If so, use that as the word count target
instead of the default.

Check for a project-level config file at `gate_config.md` or `.gate_config` in the
working directory or manuscript directory. If found, read it — it may override
thresholds. See `references/gate-thresholds.md` for the config file format.

Record: `manuscript_path`, `word_target`, `emdash_target`, etc.

---

## Step 2 — Run All Measurements

Run every measurement using Bash or Grep. Do not estimate or infer — measure the
actual file. All measurements use `manuscript_path` from Step 1.

### 2a. Word count

```bash
# Total word count (all content including headers and references)
wc -w < "[manuscript_path]"

# Body-only word count (exclude ## headers and blank lines — closer to journal word count)
grep -v "^#" "[manuscript_path]" | grep -v "^[[:space:]]*$" | wc -w
```

Use the **body-only count** as the primary measure. Record both.

Default target: ≤ 12,000 words (body-only). Severity: **CRITICAL**.

### 2b. Em-dash frequency

```bash
# Count em-dash characters (—, Unicode U+2014)
grep -o "—" "[manuscript_path]" | wc -l
```

Compute the ratio: em-dash count ÷ (body word count ÷ 1000) = em-dashes per 1,000 words.

Default target: ≤ 1.0 per 1,000 words (i.e., ≤ 10 em-dashes per 10,000 words).
Severity: **WARN**.

### 2c. Citation count and density

```bash
# Pandoc-style citations: [@key] or [@key1; @key2]
grep -o "\[@[^]]*\]" "[manuscript_path]" | wc -l

# Placeholder-style citations: [Smith2023] or [Smith2023a]
grep -oE "\[[A-Z][a-zA-Z]+[0-9]{4}[a-z]?\]" "[manuscript_path]" | wc -l
```

Use whichever count is larger (the manuscript uses one format or the other).
Compute citation density: citations ÷ (body word count ÷ 1,000) = citations per 1,000 words.

Default target: ≥ 10 citations per 1,000 words (i.e., ≥ 1 citation per 100 words).
Severity: **WARN** if below target; **CRITICAL** if below 50% of target.

### 2d. Abstract check

```bash
# Check for abstract section (any capitalization)
grep -icE "^#+ *abstract" "[manuscript_path]"
```

Default target: count ≥ 1. Severity: **CRITICAL**.

### 2e. Section structure

```bash
# Count top-level sections (## level)
grep -cE "^## " "[manuscript_path]"

# Count subsections (### level)
grep -cE "^### " "[manuscript_path]"

# Check if Introduction has subsections (it should not, for most journals)
# Count ### headers between ## Introduction and the next ## header
```

Default targets:
- Major sections (##): 4–12. Severity: **WARN** outside range.
- Introduction subsections: 0. Check by reading the Introduction section.
  Severity: **WARN** if Introduction has subsections.

### 2f. Figure and table callouts

```bash
# Figure callouts (any capitalization, number 1-9)
grep -icE "figure [1-9]|fig\. [1-9]|fig [1-9]" "[manuscript_path]"

# Table callouts
grep -icE "table [1-9]|tab\. [1-9]" "[manuscript_path]"
```

Default target: ≥ 1 figure callout, ≥ 1 table callout.
Severity: **WARN** if either is 0.

### 2g. Unfilled placeholders

```bash
# Check for common placeholder patterns
grep -icE "\[PLACEHOLDER\]|\[INSERT\]|\[FIXME\]|\[TODO\]|\[TBD\]|TODO:|FIXME:|XXX" "[manuscript_path]"
```

Default target: 0. Severity: **CRITICAL** if > 0.

### 2h. Broken or empty citation keys

```bash
# Empty citation keys: [@] with nothing inside
grep -cE "\[@\s*\]|\[ *\]" "[manuscript_path]"

# Placeholder citations that weren't replaced: look for [CITATION] or [REF]
grep -icE "\[CITATION\]|\[REF\]|\[cite\]|\[reference\]" "[manuscript_path]"
```

Default target: 0. Severity: **CRITICAL** if > 0.

### 2i. Consistency checks

Run these with Grep to detect seam-level issues:

```bash
# Inconsistent heading capitalization (Title Case vs sentence case)
# Count ## headings in Title Case vs sentence case — flag if both styles present
grep -E "^## [A-Z][a-z]" "[manuscript_path]" | wc -l   # sentence case
grep -E "^## ([A-Z][a-z]+ ){2,}" "[manuscript_path]" | wc -l  # multi-word Title Case

# Inconsistent citation formats (mixing [@key] and [Author20xx])
pandoc_count=$(grep -ocE "\[@[a-z]" "[manuscript_path]" | wc -l)
placeholder_count=$(grep -oE "\[[A-Z][a-zA-Z]+[0-9]{4}\]" "[manuscript_path]" | wc -l)
# If both > 0, flag as mixed format
```

Severity: **WARN** for any consistency issue detected.

### 2j. Systematic review language (NLR projects only)

First check CLAUDE.md to confirm whether this is a narrative review. Skip this check
entirely for systematic reviews or scoping reviews.

For narrative review projects, scan for four sub-patterns of SLR-genre language that
signal a genre mismatch to editors:

```bash
# 4a: exact paper-count claims about the review corpus
grep -icE "[0-9]+ (papers?|studies|articles?) in (this|the) corpus" "[manuscript_path]"
grep -icE "only [0-9]+ (papers?|studies) (attempt|achieve|integrate|report|address)" "[manuscript_path]"

# 4b: exhaustive-corpus assertions
grep -icE "(none|all) of the [0-9]+ reviewed (papers?|studies)" "[manuscript_path]"
grep -icE "absent from all [0-9]+ (papers?|studies)" "[manuscript_path]"

# 4c: meta-sentences about the review's own scope
grep -icE "this review (synthesizes|examines|covers|presents|includes) [0-9]+" "[manuscript_path]"

# 4d: "corpus" as a formal SLR boundary term
grep -ic "in this corpus\|in the corpus\|the corpus" "[manuscript_path]"
```

Record counts for each sub-pattern separately. Sum 4a + 4b + 4c as "SLR pattern count";
report 4d separately.

Default targets:
- Paper-count claims (4a): 0. Severity: **WARN**.
- Exhaustive-corpus assertions (4b): 0. Severity: **WARN**.
- Meta-sentences (4c): 0. Severity: **WARN**.
- "corpus" term (4d): ≤ 2. Severity: **WARN** if > 2.

If any target is exceeded, list the matching lines in the action list (Step 4) and
direct the user to run `/lit-deregister` to fix them.

---

## Step 3 — Build the Scorecard

Compile all measurements into a scorecard. Use this exact format:

```
╔══════════════════════════════════════════════════════════════╗
║                  LIT-GATE SCORECARD                          ║
║  Manuscript: [filename]        Date: YYYY-MM-DD              ║
╠══════════════════════════════════════════════════════════════╣
║ Check                    │ Measured    │ Target     │ Status ║
╠══════════════════════════╪═════════════╪════════════╪════════╣
║ Word count (body)        │ N,NNN words │ ≤ NN,NNN   │  PASS  ║
║ Em-dash rate             │ N.N / 1k w  │ ≤ 1.0      │  PASS  ║
║ Citation density         │ N.N / 1k w  │ ≥ 10.0     │  WARN  ║
║ Abstract present         │ YES / NO    │ YES        │  PASS  ║
║ Major sections (##)      │ N           │ 4 – 12     │  PASS  ║
║ Intro has subsections    │ YES / NO    │ NO         │  PASS  ║
║ Figure callouts          │ N           │ ≥ 1        │  WARN  ║
║ Table callouts           │ N           │ ≥ 1        │  WARN  ║
║ Unfilled placeholders    │ N           │ 0          │  PASS  ║
║ Broken citation keys     │ N           │ 0          │  PASS  ║
║ Citation format          │ consistent  │ consistent │  PASS  ║
║ NLR: SLR patterns (4a–c) │ N           │ 0          │  WARN  ║
║ NLR: corpus term (4d)    │ N           │ ≤ 2        │  PASS  ║
╠══════════════════════════╧═════════════╧════════════╧════════╣
║  VERDICT:  PASS / WARN / FAIL                                ║
╚══════════════════════════════════════════════════════════════╝
```

Note: The two NLR rows appear only when CLAUDE.md confirms this is a narrative
review project. For systematic review projects, omit both rows.

**Verdict logic:**
- Any CRITICAL check fails → **FAIL** (show in red/bold, do not proceed)
- No CRITICAL failures, but one or more WARN → **WARN** (can proceed, review warnings)
- All checks pass → **PASS**

---

## Step 4 — Produce the Action List

After the scorecard, print a prioritized action list:

**If FAIL:**
```
CRITICAL — Fix before proceeding:
  1. [check name]: [what was measured] vs [target] — [specific fix]
  2. ...

Then re-run /lit-gate to confirm.
```

**If WARN:**
```
Advisory — Review before submission:
  • [check name]: [what was measured] — [why it matters / suggested fix]

These do not block the current editing pass but should be resolved before final submission.
```

**If PASS:**
```
All checks passed. The manuscript is ready for the next step.
```

---

## Step 5 — Save the Gate Report

Save a machine-readable record to the same directory as the manuscript:

```
gate_report_YYYYMMDD.md
```

Include the full scorecard, all raw measurement values, the verdict, and the action list.
This record lets you compare gate results across manuscript versions.

---

## Step 6 — Summary to User

Print the scorecard and verdict in the terminal. Keep it compact — the full report
is in the saved file.

If the verdict is FAIL, end with: "Re-run /lit-gate after fixing the critical items."
If PASS or WARN, end with: "Report saved to [path]."

---

## Constraints

- **Never estimate.** If a Bash command fails (e.g., file not found, permission issue),
  report the error and ask the user to resolve it. Do not substitute a guess.
- **Measure the file as-is.** Do not mentally "correct" the manuscript while measuring.
  The gate is a checkpoint, not an editing pass.
- **Thresholds are defaults, not absolutes.** If the user says "our journal allows
  15,000 words", update the target before running the check — don't report a false fail.
- **One gate run = one snapshot.** Do not re-run any check after seeing its result.
  Measurements are taken once, in the order above, to produce a consistent snapshot.
