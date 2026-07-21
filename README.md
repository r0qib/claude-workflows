# document-codebase: AI-Context Documentation Workflow

A generic, reusable Claude Code Workflow that extracts durable business/functional context from arbitrary codebases, producing standardized markdown documentation for AI coding assistants.

## What it does

Given any existing codebase, `document-codebase`:

1. **Auto-detects stack** (languages, frameworks, architecture pattern)
2. **Discovers documentation units** (features, routes, or modules — auto-chosen per codebase)
3. **Incrementally re-extracts** (only units changed since last run; skips unchanged for speed)
4. **Produces three doc types** in `docs/ai-context/`:
   - `00-overview.md` — project spec (stack, architecture, unit list)
   - `features/<slug>.md` (or `routes/`, `modules/` per granularity) — per-unit functional context
   - `index.md` — manifest tracking which units were documented when

**What it documents:**
- Purpose, behavior, what it does and why — **never visual detail**
- Inputs, outputs, data models, API surface, permissions, business rules, error handling, edge cases
- One-hop dependencies (what other units this unit calls)
- Every claim cited to `file:line` in source code

**What it ignores:**
- Colors, fonts, spacing, layout, component visual style
- Hallucination — strict "cite or skip" discipline

## Design principles

- **Generic**: works on any stack, any structure, any codebase
- **Incremental**: re-run fast by skipping unchanged units (git-diff based)
- **Parallel**: topologically-batched extraction avoids context-window exhaustion
- **Non-destructive**: flagged-not-deleted outputs; human review required for removal
- **Verifiable**: every non-trivial claim has a source citation in the docs

## Installation (copy to your target repo)

```bash
cp -r skills/codebase-context-extractor <target-repo>/.claude/skills/
cp workflows/document-codebase.js       <target-repo>/.claude/workflows/
cp commands/document-codebase.md        <target-repo>/.claude/commands/
```

## Usage

In the target repository:

```
/document-codebase [maxUnits] [unitFilter]
```

**First run:**
- Auto-detects stack, proposes unit boundaries
- Extracts all units (capped at `maxUnits`, default 10, max 25)
- Writes `00-overview.md`, `index.md`, and per-unit docs

**Later runs:**
- Reads `index.md` to get the last-run commit hash
- Runs `git diff` to find changed files
- Re-extracts only stale units; skips unchanged ones (fast)
- Updates `index.md` manifest

**Options:**
- `maxUnits`: Process at most N units per run. Re-run to continue remaining units.
- `unitFilter`: Only process units whose id or title contains this string (for targeted re-runs).

## Architecture

### Three-phase Workflow

The workflow script (`.claude/workflows/document-codebase.js`) runs:

1. **Recon** — One agent inspects the repo, detects stack, proposes unit list, git-diffs to classify each unit (new/stale/unchanged/removed)
2. **Map** — One agent builds a best-effort dependency graph (grepped imports)
3. **Extract** — Topologically-batched parallel agents, one per unit, with one-hop dependency summaries only
4. **Reconcile** — Pure JS: merge new records with untouched prior records

Agents are **read-only by design**; the calling slash command handles all file rendering.

### Extraction Skill

The `codebase-context-extractor` skill (`.claude/skills/codebase-context-extractor/SKILL.md`) governs per-unit extraction:

- One unit at a time (caller orchestrates boundaries)
- Golden rules: "describe what it does, never what it looks like" + "never hallucinate — cite file:line"
- Scoped to the unit's globs only (no overlap, prevents duplicate output across parallel agents)
- Returns structured JSON (not markdown) for the calling command to render

### Output Structure

```
docs/ai-context/
├── 00-overview.md          (project spec, one-time or regenerated on stack change)
├── index.md                (manifest: lists all units + git commit hash per unit)
├── features/
│   ├── auth.md
│   ├── billing.md
│   └── ...
├── routes/                 (or features/ or modules/, depending on detected granularity)
└── modules/
```

Each unit doc:
- 12 fixed sections (purpose, behavior, models, API surface, permissions, etc.)
- Target <200 lines (progressive disclosure)
- YAML frontmatter with globs, dependencies, last-scanned commit, confidence level
- Every section cites sources as `file:line`

## Incremental re-documentation

**First run:** Extracts all units.

**Later run:** Detects git-diff changes:
- Changed files → their unit(s) marked `stale` and re-extracted
- Unchanged unit → skipped, prior record carried forward into new `index.md`
- Missing unit's source → marked `removed`, flagged for manual review

No cascading staleness: if unit A depends on unit B and B changes, only B is re-extracted.
A stays unchanged (confirmed by design to keep re-run scope bounded).

## Testing checklist

See `.claude/plans/i-am-new-to-velvety-panda.md`, section 5, for 9 concrete test scenarios:

- Pure-logic tests (topological sort correctness)
- End-to-end dry run on fixture repo
- Idempotency (run twice, no changes → second run is a no-op)
- Targeted drift (touch one file → only that unit re-extracted)
- Cap/resume (maxUnits=2 → defers rest, re-run continues)
- Removed unit (deleted source → flagged, not auto-deleted)
- Manual hallucination spot audit (sample docs, verify citations)
- Injection resilience (ignore instruction-shaped comments)
- Schema/render conformance

## Known limitations (v1)

1. **No cascading staleness.** Changed dependency does not auto-invalidate dependents (by design, to bound re-run cost).
2. **No automatic tech-stack changes.** If framework/language changes, use `forceRefresh: true` or manually delete `00-overview.md`.
3. **Dependency detection is grepped, not AST-based.** May miss circular or dynamic imports; treated as best-effort.
4. **No hallucination verifier agent.** Strict-prompting only. Recommend periodic manual spot audits of claim citations.

## Troubleshooting

**"Recon phase failed"**
- Ensure this is a git repository (`git status` works)
- Ensure Claude Code can read source files

**"No units found"**
- Repository structure doesn't match auto-detect heuristics
- Manually create/extend `docs/ai-context/00-overview.md` with unit guidance and re-run

**Units keep getting re-extracted**
- Check `index.md`'s `last_run_commit`: is it a valid git commit in this repo?
- Check git history: has HEAD moved since last run? (expected)
- If git diff is malfunctioning, manually delete `index.md` and re-run to force fresh extraction

**"Removed units" in index**
- Source code for a previously-documented unit disappeared
- Either delete the doc file or restore the source, then re-run

## Design documents

- **Plan**: `.claude/plans/i-am-new-to-velvety-panda.md` — full design, assumptions, rationale, test plan
- **Skill**: `.claude/skills/codebase-context-extractor/SKILL.md` — extraction discipline, rules, golden rules
- **Workflow script**: `.claude/workflows/document-codebase.js` — four phases, schemas, orchestration logic
- **Slash command**: `.claude/commands/document-codebase.md` — entry point, renders all output doc types
- **Deep research**: Industry patterns, academic SOTA (CodeWiki, DocAgent), best practices — informed this design

## Related

- **Research findings**: This design was informed by a deep-research pass across industry practice (Gitingest, Sourcegraph Cody, Anthropic's own multi-agent system, academic codebase-to-docs systems CodeWiki and DocAgent).
- **Reusable skill**: The `codebase-context-extractor` skill can also be invoked directly on a single unit (not just via this workflow) if needed for targeted extraction.

## License

This is a generic, open-ended tool for codebase documentation. Use it freely on any codebase, for any purpose.
