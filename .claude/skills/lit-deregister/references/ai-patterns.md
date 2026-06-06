# AI Writing Pattern Reference

Full pattern catalog for the lit-deregister skill. Organized by category.
Each entry has: detection strategy, count target, rewrite guidance, and examples.

---

## Category 1 — Antithesis Constructions

### What they are and why they're a problem

The "not X, but Y" construction signals significance by negating an alternative.
Used occasionally it is a legitimate rhetorical tool. In AI-assisted drafts it
becomes a reflex: every distinction is framed as a contrast, which makes the prose
feel melodramatic and uniform. Human reviewers notice when this structure appears
more than a few times.

### Detection patterns (search case-insensitively)

Primary forms:
- `not [word(s)], but [word(s)]`
- `not merely [word(s)], but [word(s)]`
- `not simply [word(s)], but [word(s)]`
- `not just [word(s)], but [word(s)]`
- `not only [word(s)], but [word(s)]`

Secondary forms (antithesis via semicolon or dash):
- `not [word(s)]; it is [word(s)]`
- `not [word(s)]; they are [word(s)]`
- `not [word(s)]; rather, [word(s)]`
- `not [word(s)] — [word(s)]` (where the dash introduces the positive framing)

Regex hint: `\bnot\b.{5,80}\bbut\b` catches most primary forms.

### Count target

**≤ 2 per manuscript** (regardless of word count). If the manuscript is over 10,000
words, allow ≤ 1 per 5,000 words, maximum 4 total.

### Which instances to keep (when count exceeds target)

Keep the constructions where:
1. The contrast is genuinely the most concise way to draw the distinction
2. The negated alternative is something the reader might plausibly assume

Rewrite the constructions where:
1. The "not X" part is not a real assumption the reader would hold
2. The positive statement (Y) could stand alone without losing meaning
3. The sentence is in a sequence of multiple antithesis constructions

### Rewrite strategy

The core move: **state Y directly, without negating X first**.

The "not X" qualifier almost always adds zero information. What matters is Y.

```
Before: "This framework is not a simulation tool but a clinical decision aid."
After:  "This framework serves as a clinical decision aid."

Before: "The model does not merely predict risk; it enables actionable intervention."
After:  "The model enables actionable intervention by predicting risk."

Before: "Integration is not optional but essential for achieving clinical utility."
After:  "Integration is essential for clinical utility."
```

When the contrast genuinely matters (keep these):
```
Keep: "The approach differs not in architecture but in training objective — a
       distinction that accounts for the 8-point performance gap."
Why:  The contrast is specific, the claimed consequence is stated, and the
      sentence would lose precision if rewritten as a direct statement.
```

---

## Category 2 — Editorializing Adjectives and Intensifiers

### What they are and why they're a problem

These words assert that something is significant, impressive, or unusual — instead
of letting the evidence make that case. In peer review, this reads as insecurity:
if you need to tell the reader that a result is "striking", the result is probably
not speaking for itself. Human-written academic prose demonstrates significance
through specificity, not through superlatives.

### Prohibited list (target = 0)

Remove every instance of these when used to characterize research findings,
methods, or the field's progress:

| Word / phrase | Typical AI usage to eliminate |
|---|---|
| striking / strikingly | "a striking improvement", "strikingly outperformed" |
| remarkable / remarkably | "remarkable accuracy", "remarkably consistent" |
| unprecedented | "an unprecedented advance", "unprecedented scale" |
| groundbreaking | "a groundbreaking approach" |
| particularly consequential | "a particularly consequential finding" |
| not a minor caveat | "this is not a minor caveat" |
| precisely the | "this is precisely the kind of..." |
| crucially (sentence-opener) | "Crucially, this enables..." |
| compellingly | "compellingly demonstrated" |
| transformative (when applied to methods) | "a transformative methodology" |

**Exception — do not remove if:** the word is part of a direct quotation, a
dataset name, a paper title, or a proper noun. Also keep "critical" when it has
a domain-specific technical meaning (e.g., "critical pathway", "critical point
in thermodynamics").

### Controlled-use list (target ≤ 3 per manuscript total across all items)

These are not prohibited but become problematic through overuse:

| Word | Acceptable use |
|---|---|
| important / importantly | When "important" carries a specific technical sense |
| notable / notably | When pointing to a genuinely unexpected result |
| significant | When referring to statistical significance specifically |
| powerful | When describing a quantifiable advantage |
| key | When identifying the specific variable or step that drives an outcome |
| compelling | When paired with a specific cited result |

### Rewrite strategy

**Move 1 — Delete the adjective.** Often the sentence works without it.

```
Before: "This represents a striking improvement over prior methods."
After:  "This improves over prior methods."
        [or, if a number exists:] "This improves AUC from 0.72 to 0.91 over prior methods."
```

**Move 2 — Replace with the specific evidence.** If the sentence feels thin after
deletion, the right fix is specificity, not a different adjective.

```
Before: "The results are particularly compelling."
After:  "The model achieves 91% sensitivity at 94% specificity — a combination
         that prior methods could not sustain simultaneously."
```

**Move 3 — Convert to a subordinate structure.** Sometimes the claim belongs in
a dependent clause that acknowledges rather than asserts importance.

```
Before: "Crucially, this framework enables personalized treatment planning."
After:  "Because the framework personalizes treatment planning, it adapts to
         individual patient profiles without retraining."
```

---

## Category 3 — Repetitive Structural Refrains

### What they are and why they're a problem

When an AI drafts multiple sections sequentially, it often repeats the same
sentence-opening pattern — "As discussed above, ...", "Taken together, these
findings suggest ...", "This underscores the need for ..." — without the writer
noticing, because each section was drafted in isolation. The repetition reads as
a stylistic tic that signals machine authorship.

### Detection strategy

**3a. Count sentence-opening patterns**
Collect the first 4–6 words of every sentence in the manuscript. Flag any opening
that appears 4 or more times. Common offenders in AI drafts:

- "Taken together, these findings..."
- "This underscores the need for..."
- "As discussed above, ..."
- "Collectively, these studies..."
- "These results demonstrate that..."
- "This highlights the importance of..."
- "Together, these observations suggest..."
- "Importantly, this approach..."
- "Notably, [subject] [verb]..."
- "In summary, ..."

**3b. Count multi-word phrase repetitions**
For any 3+ word phrase that appears in 4 or more separate paragraphs, flag it.
(A phrase appearing 4 times in the same paragraph is a different problem — likely
intentional parallelism.)

### Count target

- Any single sentence-opening pattern: ≤ 3 occurrences per manuscript
- Any 3+ word phrase: ≤ 3 occurrences in separate paragraphs
- Transition words ("Furthermore", "Moreover", "Additionally"): ≤ 5 per manuscript total
  across all three words combined

### Rewrite strategy

**Vary the grammatical entry point.** English sentences can open many ways:

| Starting type | Example |
|---|---|
| Subject-verb | "The model achieves..." |
| Participial phrase | "Trained on 10,000 slides, the model..." |
| Prepositional phrase | "Across three external cohorts, ..." |
| Subordinate clause | "Although external validation was limited to two sites, ..." |
| Adverbial | "Consistently across datasets, ..." |
| Absolute construction | "Performance aside, the key contribution is..." |

When a refrain appears 5 times and the target is ≤ 3, keep the 3 best-placed
instances and rewrite the other 2.

```
Before (5× "Taken together, these findings suggest"):
  Para 3: "Taken together, these findings suggest that multimodal fusion improves survival prediction."
  Para 5: "Taken together, these findings suggest that foundation models generalize across institutions."
  Para 7: "Taken together, these findings suggest a role for spatial context in prognosis."
  Para 9: "Taken together, these findings suggest further investigation is warranted."  ← rewrite
  Para 11:"Taken together, these findings suggest clinical translation remains a challenge." ← rewrite

After (para 9): "The pattern across studies warrants further investigation."
After (para 11):"Clinical translation remains a challenge that these studies leave incompletely addressed."
```

---

## Category 4 — Systematic Review Language in a Narrative Review

### What it is and why it's a problem

A narrative review synthesizes arguments; a systematic review enumerates evidence.
When an AI assistant drafts a narrative review, it often adopts systematic review
conventions: counting the papers it retrieved, asserting exhaustive coverage
("none of the 280 reviewed studies…"), and inserting meta-sentences that describe
the review's own scope ("this review synthesizes 280 studies…"). These patterns
signal to editors that the paper was drafted by someone who conflated the two
genres — or that the AI did.

The fix is not to suppress accurate information; the underlying facts are often
true. The fix is to restate them in the register of a narrative review: qualitative
scope descriptors, field-level claims, and gap language that names the absence
without invoking a bounded corpus count.

### Detection patterns

**4a. Exact paper-count claims about the review corpus**

Search for any sentence that states a number followed by a paper-class noun followed
by a corpus-locating phrase:

- `[N] (papers?|studies|articles?) in (this|the) corpus`
- `[N] (papers?|studies) in this review`
- `only [N] (papers?|studies) (attempt|achieve|integrate|report|address)`
- `[N] pathomic|radiogenomics|foundation model papers`
- `the [N] (papers?|studies) that (form|constitute|make up)`

Regex hint: `\b\d+\b.{0,40}(papers?|studies|articles?).{0,40}(corpus|review|cluster)`

**4b. Exhaustive-corpus assertions**

Sentences that assert a finding holds across all papers in a bounded set:

- `(none|all) of the [N] reviewed (papers?|studies)`
- `absent from all [N] (papers?|studies)`
- `across all [N] (papers?|studies) reviewed`
- `(zero|no) (paper|study) in (this|the) corpus (reports?|addresses?|documents?)`

**4c. Meta-sentences describing the review's own scope**

- `this review (synthesizes|examines|covers|presents|includes) [N] (papers?|studies)`
- `[N] (papers?|studies) published between [YYYY] and [YYYY]`
- opening sentences of the abstract or introduction that describe the paper collection
  as a numbered dataset

**4d. "corpus" as a formal SLR boundary term**

- `in (this|the) corpus` (especially repeated ≥ 3 times)
- `the corpus` used as a definite article for a bounded paper set
- `(largest|smallest|only|most) .{0,30} in (this|the) corpus`

### Count targets

| Pattern | Target |
|---|---|
| Exact paper-count claims (4a) | 0 — rewrite all |
| Exhaustive-corpus assertions (4b) | 0 — rewrite all |
| Meta-sentences about review scope (4c) | 0 — rewrite all |
| "in this corpus" / "the corpus" as SLR term (4d) | ≤ 2 — rewrite excess |

The ≤ 2 tolerance for (4d) covers cases where "corpus" appears in a quoted paper
title or as a genuine linguistic corpus reference, not as a SLR boundary term.

### Rewrite strategy

The core move: **replace the count with a qualitative scope descriptor or a
field-level claim, and relocate the corpus-boundary information to a methods note
(if needed at all).**

**Move 1 — Qualitative scope descriptors**

Replace a paper count with a descriptor that conveys relative scale:

```
Before: "89 studies in this corpus integrate pathology with genomics"
After:  "the dominant evidence base integrates pathology with genomics"

Before: "only 12 of the 280 reviewed papers attempt three-modality fusion"
After:  "relatively few published studies attempt three-modality fusion"

Before: "the 30 digital twin papers in this corpus"
After:  "digital twin papers in the reviewed literature"
```

**Move 2 — Field-level gap language**

Replace "no study in the corpus" exhaustive assertions with absence claims
that refer to the published literature rather than a bounded count:

```
Before: "None of the 280 reviewed studies reports a prospective clinical trial result"
After:  "The reviewed literature contains no prospective clinical trial results"

Before: "Prospective validation is absent from all 280 papers in the corpus"
After:  "Prospective validation is absent from the reviewed literature"

Before: "no study in the corpus documents a change in patient management"
After:  "no published study documents a change in patient management"

Before: "federated multimodal learning for cancer is entirely absent from this corpus"
After:  "federated multimodal learning for cancer has not been reported in the published literature"
```

**Move 3 — Remove or integrate meta-sentences**

Meta-sentences that describe the paper collection as a numbered dataset should
be removed or folded into a scope description without the count:

```
Before: "This review synthesizes 280 studies published between 2019 and 2026 that
         integrate at least two of the following modalities: WSI, radiology, and omics."
After:  "The studies surveyed here span the intersection of whole-slide histopathology,
         cross-sectional radiology, and molecular omics data, integrating at least two
         of these modality classes within a single analytical framework."
```

**Move 4 — Replace "in this corpus" / "the corpus"**

```
Before: "the largest and most methodologically mature segment of this review's corpus"
After:  "the largest and most methodologically mature segment of the reviewed literature"

Before: "Four major architectural strategies appear in the corpus"
After:  "Four major architectural strategies appear in the literature"

Before: "the most sparsely represented integration space in the corpus"
After:  "the most underrepresented integration space in the published literature"
```

### What NOT to change

- Numbers that belong to a specific cited study: patient counts, AUC values, cohort
  sizes. These are auditable empirical claims. ✓ "validated on 2,279 patients across
  12 centers" is always correct.
- Counts of 2–4 that appear in the conclusion or abstract as a structural summary
  (e.g., "Three barriers limit clinical translation") — these enumerate arguments,
  not papers.
- "corpus" when it refers to a text corpus in the NLP sense, or appears in a paper
  title or quoted phrase.

---

## Category 5 — Other AI Register Markers (Flag Only — Do Not Mass-Rewrite)

These patterns are worth noting but do not require systematic rewriting unless
they are extremely frequent (> 5 instances per 10,000 words). Flag them in the
report but do not rewrite unless specifically asked.

| Pattern | Why it signals AI | Threshold to flag |
|---|---|---|
| "It is worth noting that" | Metatextual commentary on the note's value | > 3 |
| "It is important to emphasize" | Author tells reader what to think | > 3 |
| "plays a crucial role in" | Vague attribution of importance | > 4 |
| "has emerged as" | Vague field-level claim | > 4 |
| "paves the way for" | Cliché future-direction phrase | > 3 |
| "holds great promise" / "shows great promise" | Unsupported optimism | > 2 |
| "state of the art" / "state-of-the-art" | Check for specificity; OK if citing a benchmark | > 6 |

For each flagged pattern, include a count in the report under a "Flagged for
monitoring" section, but do not rewrite unless the user asks.

---

## Quick Reference — Rewrite Decision Tree

```
Is the sentence making an auditable factual claim?
  YES → Do not rewrite the substance; only change the register framing
  NO  → Proceed to rewrite

Is the problematic word a technical term in this field?
  YES → Keep it; note in "Patterns Kept" section
  NO  → Proceed to rewrite

Is this an antithesis construction?
  COUNT ≤ 2 → Keep it if it is genuinely the best framing
  COUNT > 2 → Rewrite to direct positive statement

Is this an editorializing adjective on the prohibited list?
  YES (any count) → Delete or replace with specificity
  NO (controlled list, count ≤ 3) → Keep

Is this a refrain?
  COUNT ≤ 3 → Keep
  COUNT > 3 → Rewrite excess instances using a different grammatical structure
```
