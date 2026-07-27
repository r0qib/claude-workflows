---
description: Scaffold a docs tree and extract durable AI-context (business + UI behavior) from this codebase
argument-hint: "[natural language — e.g. 'use 5 agents, just the auth module', or nothing]"
---

# /document-codebase

Scaffold a fixed documentation folder tree (create-if-missing) and document this
codebase's business/functional logic and UI behavior into `docs/` for AI coding
assistants — what each module/feature does, why, and how the user experiences it.
Never visual styling; no line-number pinning.

## How it works

**Scaffold-then-fill:** Creates a predictable folder tree once, then fills only the
`modules/` and `technical/` docs. The open folders (`business/`, `system/`,
`project-management/`, `testing/`) get a single instructional stub and are left for you.

**Create-if-missing:** Never overwrites or deletes anything that already exists —
including a real `.claude/` in your repo, your edited docs, or the stubs.

**Vertical slice:** `modules/<module>/<feature>/…` — module first, then features,
going deeper only when a feature needs it.

**Incremental:** Modules unchanged since the last run are skipped (git-diff based).

**Dynamic & natural-language controlled:** You say how many agents to use and what to
focus on in plain language; the command resolves it to concrete workflow args.

## Usage

```
/document-codebase [plain-English instructions]
```

Everything is optional. Examples:

- `/document-codebase` — document everything, auto agent count.
- `/document-codebase use about 5 agents` — cap concurrency at ~5.
- `/document-codebase just the auth and billing modules` — focus filter.
- `/document-codebase only the payments feature, 3 agents` — focus + concurrency.
- `/document-codebase re-scan everything from scratch` — force full refresh.
- `/document-codebase put the docs under .ai` — override docs root.

## Step 1 — Parse the instruction into args (natural language)

Read `$ARGUMENTS` and derive this arg object. Apply these rules; when the text is
silent on a field, use the default.

| Arg | Type | Default | Derive from phrases like… |
|---|---|---|---|
| `agentBudget` | int or `null` | `null` (auto = `min(16, cores-2)`) | "use N agents", "spawn N workers", "keep it to N", "as many as needed" → `null` |
| `focus` | string[] or `null` | `null` (everything) | "just/only the X module", "focus on X and Y", "the X feature" → `["x","y"]` (lowercased). Matches module/feature **id or title**, substring. |
| `docsRoot` | string | `"docs"` | "put docs under Z", "into Z/" → `"z"` |
| `aiTools` | string[] | `["claude"]` | "scaffold for cursor too", "all AI tools" → e.g. `["claude","cursor","copilot"]` |
| `forceRefresh` | bool | `false` | "from scratch", "re-scan everything", "ignore the cache" → `true` |
| `repoPath` | string | `"."` | rarely set; only if a different path is named |

If a number is ambiguous ("a few agents"), pick a sensible small integer (e.g. 3-5) and
say what you chose in Step 2. Never invent a `focus` that wasn't asked for — default to
`null` (document everything).

## Step 2 — Echo the resolved plan

Before invoking, state the resolved plan in one line so a misparse is visible, e.g.:

> Resolved: focus = auth, billing · concurrency = 4 · docs → `docs/` · tools → `.claude` · refresh = no

You are unattended by default — after echoing, **proceed** (do not block for confirmation).

## Step 3 — Invoke the workflow

```
Workflow({
  scriptPath: ".claude/workflows/document-codebase.js",
  args: { repoPath: ".", docsRoot: <docsRoot>, aiTools: <aiTools>,
          agentBudget: <agentBudget>, focus: <focus>, forceRefresh: <forceRefresh> }
})
```

The workflow returns:
`{ docsRoot, headCommit, isFirstRun, stackSummary, technicalDocPlan, scaffold,
   extractedModules, skippedUnchanged, removedModuleIds, deferred, resolved, stats }`

## Step 4 — Scaffold the folder tree (create-if-missing)

Using `result.scaffold` and `result.docsRoot` (call it `ROOT`). **For every path below,
check existence first — if it exists, skip it untouched. Never overwrite.**

**4a. AI-tool skeletons.** For each dir in `scaffold.aiToolDirs` (e.g. `.claude`), create
`ROOT/<dir>/<sub>/` for each `sub` in `scaffold.aiToolSubdirs`
(`agents`, `commands`, `skills`, `workflows`). If a subdir would be empty, drop a
`.gitkeep` in it. If `ROOT/<dir>` already exists, leave the whole thing alone.

**4b. Open stub folders.** For each entry in `scaffold.stubFolders`
(`business`, `system`, `project-management`, `testing`), create `ROOT/<path>/` and, only
if it does not already exist, `ROOT/<path>/README.md` with:

```markdown
# <title>

Organize your <path>-related content here.

_This folder was scaffolded by /document-codebase and intentionally left open.
Nothing here is auto-generated — it's yours to fill._
```

**4c. Real target folders.** Ensure `ROOT/modules/` and `ROOT/technical/` exist.

## Step 5 — Write `00-overview.md`

Write `ROOT/00-overview.md` only if this is the first run OR `stackSummary` is non-null.
If it exists, preserve any hand-written prose in the body; only refresh the frontmatter
and the auto sections.

```markdown
---
doc_type: overview
generated_by: document-codebase workflow
last_scanned_commit: <headCommit>
updated: <ISO date, now>
primary_languages: <stackSummary.primaryLanguages>
frameworks: <stackSummary.frameworks>
architecture_pattern: <stackSummary.architecturePattern>
---

# <Repo Name> — AI Context Overview

> Durable functional/business + UI-behavior context for AI coding assistants.
> Excludes visual styling. Regenerated incrementally — prose in the body survives re-runs.

## 1. Purpose
[hand-written or preserved]

## 2. Detected Stack
- **Primary languages:** <stackSummary.primaryLanguages>
- **Frameworks:** <stackSummary.frameworks>
- **Architecture:** <stackSummary.architecturePattern>

## 3. Entry Points
- <stackSummary.entryPoints[...]>

## 4. Modules
See `index.md` for the full module → feature list.

## 5. Notes & Open Questions
[hand-written or preserved]
```

## Step 6 — Write module & feature docs

For each `module` in `extractedModules`:

**6a. Module overview** → `ROOT/modules/<module.id>/_module.md`:

```markdown
---
doc_type: module
module_id: <module.id>
title: <module.title>
paths: <module.globs>
last_scanned_commit: <headCommit>
confidence: <module.overview.confidence>
---

# <module.title>

> <module.overview.oneLineSummary>

## Purpose
<module.overview.purpose>

## Behavior
<module.overview.behavior>

## Primary Users
<module.overview.primaryUsers or "Not identified.">

## Features
<for each feature in module.features: - [<feature.title>](<feature.id>/<feature.id>.md) — <feature.record.oneLineSummary>>

## Key Files
<module.overview.keyReferences as `path` or `path::symbol` bullets; else "None identified.">

## Open Questions
<module.overview.openQuestions; else "None found.">
```

**6b. Each feature** → `ROOT/modules/<module.id>/<feature.id>/<feature.id>.md`.
(If the extractor marked deeper sub-features, nest them the same way one level down.)

```markdown
---
doc_type: feature
module_id: <module.id>
feature_id: <feature.id>
title: <feature.title>
paths: <feature.globs>
depends_on: <feature.record.dependsOn>
last_scanned_commit: <headCommit>
confidence: <feature.record.confidence>
---

# <feature.title>

> <feature.record.oneLineSummary>

## 1. Purpose & Primary Users
<record.purpose>

Primary users: <record.primaryUsers>

## 2. Behavior
<record.behavior>

## 3. User Flows
<record.userFlows as: **<flow>** then an ordered list of steps; else "None identified.">

## 4. Triggers & Inputs
<record.triggersAndInputs as bullets (trigger / input / source if present); else "None identified.">

## 5. Data Models
<record.dataModels as bullets (name / fields / relationships / source if present); else "None identified.">

## 6. API Surface
<record.apiSurface as bullets (method+path or signature / request / response / errors / source if present); else "None identified.">

## 7. Business Rules
<record.businessRules as bullets (rule; Given/When/Then when present; source if present); else "None identified.">

## 8. Permissions & Authorization
<record.permissions as bullets (gate / condition / source if present); else "None identified.">

## 9. Error Handling
<record.errorHandling as bullets (errorType / trigger / userVisibleOutcome / source if present); else "None identified.">

## 10. States & Edge Cases
<record.statesAndEdgeCases as bullets (condition / behavior); else "None identified.">

## 11. Dependencies
<record.dependsOn as bullets (one-hop unit/module names); else "None identified.">

## 12. Key Files
<record.keyReferences as `path` or `path::symbol` bullets; else "None identified.">

## 13. Open Questions
<record.openQuestions as bullets; else "None found.">
```

> No "Source Index" section, and no line numbers anywhere. References are file paths or
> `path::symbol` only.

## Step 7 — Write technical docs

For each entry in `technicalDocPlan`, write `ROOT/technical/<id>.md` (create-if-missing;
if it exists, leave it) with a frontmatter (`doc_type: technical`, `title`,
`last_scanned_commit`) and a body seeded from the entry's `purpose` plus the relevant
`stackSummary` fields (`buildAndRun`, `conventions`, `entryPoints`, `architecturePattern`).
These are project-level docs, not per-module.

## Step 8 — Write `index.md`

Write `ROOT/index.md`:

```markdown
---
doc_type: index
last_run_commit: <headCommit>
last_run_at: <ISO date, now>
modules:
  - id: <module.id>
    title: <module.title>
    doc: modules/<module.id>/_module.md
    summary: <module.overview.oneLineSummary>
    features: <list of feature ids>
    last_scanned_commit: <headCommit>
    status: documented
  [... for each module in extractedModules]
  [... PLUS carry forward all prior modules listed in skippedUnchanged from the old index.md]
---

# AI Context Index

Updated <ISO date, now>.

| Module | Summary | Features | Last Scanned | Doc |
|--------|---------|----------|--------------|-----|
<row per module: id | summary | feature count | short SHA (first 7) | [view](modules/<id>/_module.md)>

## Removed — pending human review

<for each id in removedModuleIds: - **<id>**: `modules/<id>/` — source no longer matches its globs; delete manually if no longer needed.>
```

Carry forward unchanged modules' rows from the previous `index.md` untouched.

## Step 9 — Final report

```
✓ Documentation run complete.

Resolved plan: concurrency = <resolved.agentBudget> · focus = <resolved.focus or "all"> · docs → <docsRoot>/ · tools → <resolved.aiToolDirs>

Scaffolded (created if missing): <docsRoot>/{<aiToolDirs>, business, system, project-management, technical, testing}/
Modules documented this run: <stats.extractedThisRun> (<stats.featuresExtracted> features)
Unchanged (carried forward): <stats.unchanged>
Deferred by focus filter: <stats.deferred>
Removed (flagged): <stats.removed>

Output: <docsRoot>/
- 00-overview.md, index.md
- modules/<id>/_module.md + <feature>/<feature>.md
- technical/<id>.md

<if removedModuleIds.length> ⚠️  See "Removed — pending human review" in index.md </if>
<if any Step-1 defaults were assumed from ambiguous input, note them here.>
```

## Troubleshooting

**"Recon phase failed"** — Not inspectable. Is this a git repo (`git status` works)? Can
Claude Code read the source files?

**"No modules identified"** — Recon couldn't partition the repo. Add a note in
`docs/00-overview.md` describing your module boundaries and re-run.

**Modules keep getting re-extracted** — Check `index.md`'s `last_run_commit` is a valid
commit in this repo. If git diff misbehaves, delete `index.md` and re-run for a fresh pass.

**A real `.claude/` (or edited doc) got changed** — It should not. Scaffolding is
create-if-missing; report this as a bug if you see an existing file overwritten.
