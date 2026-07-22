# document-codebase: AI-Context Documentation Workflow[^1]

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

This is a portable, drop-in tool — three files, no dependencies beyond Claude Code + git.

```bash
cp -r .claude/skills/codebase-context-extractor <target-repo>/.claude/skills/
cp .claude/workflows/document-codebase.js       <target-repo>/.claude/workflows/
cp .claude/commands/document-codebase.md        <target-repo>/.claude/commands/
```

(See `INSTALL.md` for the same steps spelled out individually.)

No config file, no per-repo setup — the workflow auto-detects stack and structure on first run.

## Usage

In the target repository, run:

```
/document-codebase [maxUnits] [unitFilter]
```

Both arguments are optional and positional — `maxUnits` first, `unitFilter` second.

| Arg | Type | Default | Meaning |
|---|---|---|---|
| `maxUnits` | integer | `10` (hard cap `25`) | Max number of new/stale units to extract **this run**. Anything beyond the cap is deferred, not skipped — it's picked up automatically on your next run. |
| `unitFilter` | string | none (no filter) | Only consider units whose `id` or `title` **contains** this substring. Plain substring match — not a glob or regex. Applied *before* `maxUnits`, so it narrows the candidate pool first, then caps within that. |

### Common invocations

```
/document-codebase
```
Default run: detect/refresh up to 10 new-or-stale units, no filter. This is what you want for routine re-runs on a repo you've already documented once.

```
/document-codebase 25
```
Raise the ceiling to the max (25) for a big push on a large repo — still bounded, just less deferral.

```
/document-codebase 5 auth
```
Only touch units related to "auth" (matches `id` or `title`), capped at 5. Use this for a targeted re-doc of one area without disturbing the rest of the manifest.

```
/document-codebase 25 payment
```
Filter to "payment"-related units, but don't cap below what the filter already found (up to 25).

### Resuming an incomplete run (the case you'll hit on large repos)

If a run's candidate unit count exceeds `maxUnits`, the extra units are **not lost** — they're recorded in the result as `deferred` and reported to you at the end:

```
Deferred to next run: 14 units
(run /document-codebase again to continue)
```

To pick up where you left off, run the exact same command again:

```
/document-codebase
```

Each run re-reads `index.md`'s `last_run_commit`, git-diffs from there, and re-classifies every previously known unit as `stale`/`unchanged`/`removed` — units that were merely *deferred* (not yet extracted, source unchanged) will simply be re-proposed as candidates and picked up in commit-hash order until the whole backlog is documented. No flags needed to "resume" — deferral + re-run is the resume mechanism. If you want to fast-forward through a large backlog, just bump `maxUnits` on the next call instead of running the default repeatedly.

**First run on a repo:**
- Auto-detects stack, proposes unit boundaries (feature/route/module — chosen automatically per codebase)
- Extracts up to `maxUnits` units; defers the rest
- Writes `00-overview.md`, `index.md`, and per-unit docs under `docs/ai-context/`

**Later runs on the same repo:**
- Reads `index.md` to get the last-run commit hash
- Runs `git diff` against it to find changed files
- Re-extracts only `stale` units (touched files) and any still-`deferred` units; skips `unchanged` ones entirely (fast, idempotent)
- Updates `index.md` manifest, carrying forward unchanged units untouched

### How it partitions a codebase

The Recon phase picks the unit granularity per repo, not globally — it looks for:
- **feature/domain** boundaries if there's an obvious `features/`, `pages/`, or `domains/` directory
- **route/endpoint** boundaries if routes are centralized and orthogonal (e.g. Express `routes/`)
- **module/top-level** as the fallback for anything else

Each unit gets a stable kebab-case `id`, a `title`, a `type`, and a list of file `globs` that define its read scope. This choice (and the rationale for it) is written into `00-overview.md` on first run — if it picks a granularity you disagree with, edit `00-overview.md`'s unit guidance and re-run rather than fighting the auto-detection.

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

- **Skill**: `.claude/skills/codebase-context-extractor/SKILL.md` — extraction discipline, rules, golden rules
- **Workflow script**: `.claude/workflows/document-codebase.js` — four phases, schemas, orchestration logic
- **Slash command**: `.claude/commands/document-codebase.md` — entry point, renders all output doc types

## Related

- **Research findings**: This design was informed by a deep-research pass across industry practice (Gitingest, Sourcegraph Cody, Anthropic's own multi-agent system, academic codebase-to-docs systems CodeWiki and DocAgent).
- **Reusable skill**: The `codebase-context-extractor` skill can also be invoked directly on a single unit (not just via this workflow) if needed for targeted extraction.

## License

This is a generic, open-ended tool for codebase documentation. Use it freely on any codebase, for any purpose.

---

> Built as R&D at Brain Station 23 PLC, exploring how existing codebases can be brought to AI-DLC readiness — durable, machine-readable context that lets AI coding assistants work on a codebase without re-deriving it from scratch each time.
