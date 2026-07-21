---
description: Extract durable AI-context docs from this codebase for coding assistants
argument-hint: "[maxUnits] [unitFilter]"
---

# /document-codebase

Document this codebase's business/functional logic into `docs/ai-context/` for AI
coding assistants — what each unit does and why, never what it looks like.

## How it works

**Incremental:** Units unchanged since the last run are skipped (fast idempotent operation).

**Parallel:** Changed units are documented in topologically-batched parallel extraction
to avoid context-window exhaustion.

**Safe:** No hallucinations — every claim is cited to file:line in source.

## Usage

```
/document-codebase [maxUnits] [unitFilter]
```

- `maxUnits`: Process at most N units per run (default 10, max 25). Use smaller values
  to review docs between batches.
- `unitFilter`: Only process units whose id or title contains this string.

## Before you run

If `docs/ai-context/index.md` already exists, the command will:
1. Read it to detect the last-run commit hash.
2. Use `git diff` to identify changed units.
3. Preserve unchanged units' rows in the output index.

## Running the workflow

<workflow>

1. If `docs/ai-context/index.md` exists, read it now so we can carry forward unchanged units later.

2. Invoke the workflow:

Workflow({
  scriptPath: ".claude/workflows/document-codebase.js",
  args: { repoPath: ".", maxUnits: ${1:-10}, unitFilter: "${2:}" }
})

3. The workflow returns: `{ isFirstRun, headCommit, stackSummary, extracted, skippedUnchanged, removedUnitIds, deferred, stats }`

</workflow>

## Rendering output

Based on the workflow result:

**Step 1: Write or update `00-overview.md`** (only if this is the first run OR
`stackSummary` is non-null):

```markdown
---
doc_type: overview
generated_by: document-codebase workflow
last_scanned_commit: <headCommit>
updated: <ISO date, now>
unit_granularity: <stackSummary.unitGranularity>
primary_languages: <stackSummary.primaryLanguages>
frameworks: <stackSummary.frameworks>
architecture_pattern: <stackSummary.architecturePattern>
---

# <Repo Name> — AI Context Overview

> Purpose: Durable functional/business context for AI coding assistants. Excludes
> UI/visual detail. Regenerated incrementally — do not hand-edit frontmatter; prose
> in the body survives re-runs.

## 1. Purpose

[hand-written or preserved from prior run]

## 2. Detected Stack

- **Primary languages:** <stackSummary.primaryLanguages>
- **Frameworks:** <stackSummary.frameworks>
- **Architecture:** <stackSummary.architecturePattern>

## 3. Architecture

[hand-written or preserved from prior run]

## 4. Entry Points

- <stackSummary.entryPoints[0]>
- ...

## 5. Documentation Units

See `index.md` for the full unit list.

## 6. Notes & Open Questions

[hand-written or preserved from prior run]
```

**Step 2: For each unit in `extracted`, write `docs/ai-context/<unit.type>/<unit.id>.md`:**

```markdown
---
doc_type: unit
unit_id: <unit.id>
unit_type: <unit.type>
title: <unit.title>
paths: <unit.globs>
depends_on: <record.dependsOn>
last_scanned_commit: <headCommit>
confidence: <record.confidence>
---

# <unit.title>

> <record.oneLineSummary>

## 1. Purpose & Primary Users

<record.purpose>

Primary users: <record.primaryUsers>

## 2. Behavior

<record.behavior>

## 3. Triggers & Inputs

<record.triggersAndInputs as bullet list, each with source citation>

If none: "None identified."

## 4. Data Models

<record.dataModels as bullet list, each with name, fields, relationships, source citation>

If none: "None identified."

## 5. API Surface

<record.apiSurface as bullet list, each with method/path/signature, request/response shape, error codes, source citation>

If none: "None identified."

## 6. Business Rules

<record.businessRules as bullet list, each with rule/given/when/then, source citation>

If none: "None identified."

## 7. Permissions & Authorization

<record.permissions as bullet list, each with gate/condition, source citation>

If none: "None identified."

## 8. Error Handling

<record.errorHandling as bullet list, each with error type/trigger/outcome, source citation>

If none: "None identified."

## 9. States & Edge Cases

<record.statesAndEdgeCases as bullet list, each with condition/behavior>

If none: "None identified."

## 10. Dependencies

One-hop only (see step 10 in this file's frontmatter `depends_on`):

<for each id in record.dependsOn:>
- <id>: [see docs/ai-context/<type>/<id>.md]

If none: "None identified."

## 11. Open Questions

<record.openQuestions as bullet list>

If none: "None found."

## 12. Source Index

<deduplicate all source citations from steps above, render as path:line or path:line-line>

If none: "No citations."
```

**Step 3: Write or update `docs/ai-context/index.md`:**

```markdown
---
doc_type: index
last_run_commit: <headCommit>
last_run_at: <ISO date, now>
units:
  - id: <u.id>
    title: <u.title>
    type: <u.type>
    doc: <type>/<id>.md
    summary: <oneLineSummary>
    depends_on: <record.dependsOn>
    last_scanned_commit: <headCommit>
    status: documented
  [... for each unit in extracted]
  [... PLUS carry forward all prior units from the old index.md that appear in skippedUnchanged]
---

# AI Context Index

All documentation units in this codebase (updated <ISO date, now>):

| Unit | Type | Summary | Depends On | Last Scanned | Doc |
|------|------|---------|-----------|----------------|-----|
<for each unit in frontmatter.units>
| <unit.id> | <unit.type> | <unit.summary> | <unit.depends_on join with ', '> | <unit.last_scanned_commit short SHA (first 7 chars)> | [view](<unit.doc>) |
...

## Removed — pending human review

The following units' source code no longer exists (no files matched their globs).
Their doc files remain on disk; delete manually if no longer needed:

<for each id in removedUnitIds:>
- **<id>** (was: <prior title from old index, or "unknown">): `docs/ai-context/<type>/<id>.md`

To proceed, either delete the doc file and re-run this command, or restore the
source code for the unit.
```

## Final report

Report to the user:

```
✓ Documentation complete!

Extracted this run: <stats.extractedThisRun> units
Unchanged (carried forward): <stats.unchanged> units
Total documented: <stats.extractedThisRun + stats.unchanged> units

Deferred to next run: <stats.deferred> units
(run /document-codebase again to continue)

Output: docs/ai-context/
- 00-overview.md
- index.md
- <type>/<id>.md (x<stats.extractedThisRun>)

<if removedUnitIds.length > 0>
⚠️  Removed units: see "Removed — pending human review" in index.md
</if>
```

## Troubleshooting

**"Recon phase failed"** — The workflow couldn't inspect the repository. Check:
- Is this a git repository? (`git status` should work)
- Can Claude Code read your source files? (check permissions)

**"No units found"** — The workflow couldn't auto-detect a sensible unit granularity.
Manually create or extend `docs/ai-context/00-overview.md` with guidance on how to
partition your codebase, then re-run.

**"Removed units" section appears** — Source code for a previously-documented unit
is now gone. Review the list and either delete the doc file or restore the source,
then re-run.
