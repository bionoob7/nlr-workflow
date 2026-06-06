---
name: lit-status
description: >
  Session startup status check for academic literature review projects. Reads the
  project log, counts files in each workflow directory, infers the current phase,
  and prints a concise status panel — all in one shot. Replaces the manual "check
  the project before starting" habit that users frequently forget. Use this skill
  at the start of every session, when resuming after a break, when you're unsure
  where a project stands, or when a user says "let's continue the review" or "where
  were we?" without providing context. Works for any lit-review project structure.
  Invoke with /lit-status.
allowed-tools: Read Glob Grep Bash
metadata:
  version: "1.0"
---

# Lit-Status — Session Startup Status Check

Run this at the start of a session. The goal is to give the user (and yourself) a
shared, accurate picture of the project state in under 60 seconds, based on what
the files actually say — not what you remember from earlier in the conversation.

This skill is read-only. It does not modify any files.

---

## Step 1 — Locate the Project Root

The project root is the directory that contains the main workflow subdirectories.
Check these in order:

1. The current working directory (if it contains LOG.md, CLAUDE.md, or ≥ 3 of the
   expected workflow directories)
2. Parent directories up to 2 levels up
3. If the user named a path in their message, use that

**Expected workflow directories** (look for any combination of these names):

| Purpose | Common directory names |
|---|---|
| Search results | `search/`, `results/`, `data/`, `raw/` |
| Screening | `screening/`, `screen/`, `filtered/` |
| Extraction notes | `extractions/`, `notes/`, `papers/`, `summaries/`, `readings/` |
| Synthesis / themes | `synthesis/`, `analysis/`, `themes/`, `clusters/` |
| Outline | `outline/`, `outlines/`, `structure/` |
| Draft sections | `draft/`, `drafts/`, `sections/`, `writing/` |
| Final manuscript | `manuscript/`, `output/`, `final/`, `ms/` |
| References | `references/`, `bib/`, `library/` |

If you cannot locate a project root with ≥ 3 recognizable directories, tell the user
and ask them to confirm the working directory before continuing.

---

## Step 2 — Read the Project Log

Look for a session log file: `LOG.md`, `log.md`, `SESSIONS.md`, or `journal.md` in
the project root.

If found, read the **last entry only** (the most recent `## YYYY-MM-DD` block or
equivalent). Extract:
- Date of last session
- What was done (`Done:` line or equivalent)
- What was planned next (`Next:` line or equivalent)

If no log file exists: note "No log found — this may be the first session."

---

## Step 3 — Count Files in Each Directory

For each directory found in Step 1, count the relevant files:

```bash
# Count .md files (extraction notes, outlines, drafts)
ls -1 [dir]/*.md 2>/dev/null | wc -l

# Count .csv files (search results, screening lists)
ls -1 [dir]/*.csv 2>/dev/null | wc -l

# Count .pdf files (source papers)
ls -1 [dir]/*.pdf 2>/dev/null | wc -l

# Count .bib files (references)
ls -1 [dir]/*.bib 2>/dev/null | wc -l
```

Also check for a manuscript file with quick word count:

```bash
# If a manuscript file exists, get word count
wc -w [manuscript_file] 2>/dev/null
```

For the screening directory specifically, try to extract the number of included papers:

```bash
# Count rows with Score=2 (included) in any CSV
grep -c ",2," [screening_dir]/*.csv 2>/dev/null | tail -1
```

If that fails, use the extraction note count as a proxy for "papers processed."

---

## Step 4 — Infer the Current Phase

Based on what files exist and how complete they are, assign the most likely phase.
Use this decision logic:

```
Does a polished/v2/submitted manuscript exist?
  YES → Phase 6–7: Polish or submission preparation

Does any manuscript file exist (v1, draft, cited)?
  YES and extractions are > 80% of included papers → Phase 5–6: Writing / early polish
  YES and extractions < 80% → Unusual — flag as potential gap

Do synthesis files or outline files exist?
  YES → Phase 4–5: Synthesis / writing

Do extraction notes exist?
  YES, count ≥ 5 → Phase 3: Extraction (in progress or complete)
  YES, count < 5 → Phase 2–3: Just started extractions

Does a screening CSV exist?
  YES → Phase 2: Screening (in progress or complete)

Do only search result files exist?
  YES → Phase 1–2: Search complete, screening not started

No files in any directory → Phase 1: Not started
```

State the inferred phase as a single line: "Inferred phase: Phase N — [name]"

---

## Step 5 — Print the Status Panel

Use this exact format. Fill in real values; use `—` for directories that don't exist.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  LITERATURE REVIEW STATUS  ·  [today's date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Last session: [date, or "No log found"]
    Done: [done line from log, or "—"]
    Next: [next line from log, or "—"]

  Inferred phase: Phase N — [phase name]

  Project files:
    search/          [N files]   [✓ / ○ / —]
    screening/       [N files]   [✓ / ○ / —]
    extractions/     [N files]   [✓ / ● / ○ / —]
    synthesis/       [N files]   [✓ / ○ / —]
    outline/         [N files]   [✓ / ○ / —]
    draft/           [N files]   [✓ / ○ / —]
    manuscript/      [N files, word count if exists]
    references/      [N .bib files]

  Included papers:  [N included / N screened, or "unknown"]
  Extractions:      [N / N_included  (X% complete), or just N]
  Manuscript:       [N words, or "not started"]

  Suggested next step:
    [one concrete sentence about what to do this session]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Status symbols:**
- `✓` — directory exists and has files
- `●` — in progress (exists but incomplete relative to expected total)
- `○` — not yet started (directory exists but empty, or expected but absent)
- `—` — directory not found in this project

---

## Step 6 — Flag Any Blocking Issues

After the panel, add a short "Flags" section if any of these are true:

| Condition | Flag |
|---|---|
| Extractions < 50% of included papers but draft exists | "Draft started before extraction complete — coverage may be thin" |
| No log file | "No LOG.md found — consider creating one to track session history" |
| Manuscript word count > 15,000 | "Manuscript may exceed journal limits — check target word count" |
| `[PLACEHOLDER]` or `[TODO]` found in manuscript | "Manuscript contains unfilled placeholders" |
| Extraction count = 0 but screening shows included papers | "Included papers exist but no extraction notes started" |
| Draft directory empty but synthesis directory has files | "Synthesis exists but no draft started — ready to begin writing?" |

Only flag conditions that are actually true. Do not fabricate warnings.

---

## Step 7 — Suggested Next Step

End with one concrete suggested action for today's session, based on the phase and
what looks incomplete. Examples:

- Phase 2: "Screen [N] remaining abstracts in `borderline_review.csv`"
- Phase 3: "Extract [N] remaining included papers (currently [X]% complete)"
- Phase 4: "Draft the thematic synthesis outline — extraction is complete"
- Phase 5: "Continue drafting [section name] — [N] sections remain"
- Phase 6: "Run /lit-audit before proceeding to style editing"
- Phase 7: "Replace citation placeholders using the library.bib file"

If the log's "Next:" line is specific and matches the current state, use that instead.

---

## Constraints

- **Read-only.** Do not write or modify any files during this skill.
- **Fast.** Prioritize Bash file counts over reading file contents. Only read LOG.md
  and a small portion of the manuscript (first 50 words) if needed.
- **Honest about uncertainty.** If you cannot determine the number of included papers,
  say "unknown" rather than guessing. If the phase is ambiguous between two options,
  name both: "Phase 3–4 (extraction nearly complete, synthesis not yet started)."
- **One status panel, then stop.** Do not proactively start the session's work — just
  report status and wait for the user to direct the next action.
