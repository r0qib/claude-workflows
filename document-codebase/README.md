# document-codebase: AI-Context Documentation Workflow

A generic, reusable Claude Code Workflow that scaffolds a predictable documentation folder
tree and extracts durable business/functional context — plus UI behavior — from arbitrary
codebases, for AI coding assistants.

## What it does

Given any existing codebase, `document-codebase`:

1. **Scaffolds a fixed folder tree** under `docs/` (create-if-missing, never overwrites).
2. **Auto-detects the stack** (languages, frameworks, architecture) → `technical/`.
3. **Discovers a module → feature tree** (vertical slice) and documents each.
4. **Incrementally re-extracts** — only modules changed since the last run (git-diff based).
5. **Is controlled in natural language** — you say how many agents to use and what to focus
   on; the slash command resolves it to concrete args.

**What it documents (for `modules/` and `technical/`):**
- Purpose, behavior, and **how the user experiences it** (user flows, navigation, states)
- Inputs, outputs, data models, API surface, permissions, business rules, error handling
- One-hop dependencies (names of other units this one calls)
- Grounded in real code — references by file path or `path::symbol`

**What it ignores:**
- Visual styling — colors, fonts, spacing, layout, component look
- Hallucination — unsupported claims become open questions, not invented facts
- **Line numbers** — references are paths/symbols, which don't rot on the next edit

## The generated folder tree

```
docs/                          (docsRoot — override with "put docs under .ai")
├── .claude/                   AI-tool skeleton (agents/ commands/ skills/ workflows/)
│                              — real dot-dir; add .cursor/ .github/ via aiTools
├── business/                  STUB — instructional README, left open for you
├── modules/                   REAL — vertical slice, the main extraction target
│   └── <module>/
│       ├── _module.md         module overview + feature list
│       └── <feature>/
│           └── <feature>.md   functional / business / UI-behavior context
├── system/                    STUB — left open
├── project-management/        STUB — left open
├── technical/                 REAL — project-level technical docs (stack, arch, build/run)
└── testing/                   STUB — left open
```

The four stub folders each get one short instructional `README.md` and are never
auto-filled. `modules/` and `technical/` hold the extracted content. Anything that already
exists — including a real `.claude/` in your repo or docs you've hand-edited — is left
untouched.

## Design principles

- **Less opinionated:** you control agent count, focus, and docs root in plain language.
- **Predictable structure:** a fixed tree, not an auto-chosen granularity.
- **Idempotent:** create-if-missing everywhere; unchanged modules skipped (git-diff).
- **Non-destructive:** removals are flagged for human review, never deleted.
- **Read-only extraction:** agents never write; the slash command renders every file.

## Installation (copy to your target repo)

Three files, no dependencies beyond Claude Code + git.

```bash
cp -r .claude/skills/codebase-context-extractor <target-repo>/.claude/skills/
cp .claude/workflows/document-codebase.js       <target-repo>/.claude/workflows/
cp .claude/commands/document-codebase.md        <target-repo>/.claude/commands/
```

(See `INSTALL.md` for the same steps spelled out.) No config file — the workflow
auto-detects stack and structure on first run.

## Usage

In the target repository:

```
/document-codebase [plain-English instructions]
```

Everything is optional and expressed in natural language. The command parses your text into
concrete workflow args and **echoes the resolved plan** before running.

| You say… | Resolves to |
|---|---|
| `/document-codebase` | everything, auto agent count |
| `use about 5 agents` | `agentBudget: 5` (concurrency cap) |
| `just the auth and billing modules` | `focus: ["auth","billing"]` |
| `only the payments feature, 3 agents` | `focus: ["payments"], agentBudget: 3` |
| `re-scan everything from scratch` | `forceRefresh: true` |
| `put the docs under .ai` | `docsRoot: ".ai"` |
| `scaffold for cursor too` | `aiTools: ["claude","cursor"]` |

### How agent count actually works

`agentBudget` is a **real concurrency cap** — the extraction work-list is chunked so no more
than that many agents run at once. Omit it and it's `auto` (`min(16, cores-2)`, the engine's
own cap). There is **no hidden per-run unit cap** (the old `maxUnits ≤ 25` is gone). If a
`focus` filter excludes some modules, they're reported as `deferred`, never silently dropped.

### First run vs. later runs

**First run:** detects stack, discovers the module→feature tree, scaffolds the folder tree,
writes `00-overview.md`, `index.md`, `modules/**`, and `technical/**`.

**Later runs:** reads `index.md`'s `last_run_commit`, `git diff`s against it, re-extracts
only `stale` modules (touched files) and any `new` ones, carries `unchanged` modules forward
untouched, and flags `removed` modules for manual review.

## Architecture

### Three-phase Workflow

`.claude/workflows/document-codebase.js`:

1. **Recon** — one agent: detect stack, discover the module→feature tree, plan technical
   docs, and git-diff to classify each module (new/stale/unchanged/removed).
2. **Extract** — a flat, dynamic `parallel()` fan-out: one read-only agent per module
   (auto-split to one per feature for `large` modules), throttled to `agentBudget`.
3. **Reconcile** — pure JS: group feature records under their module, synthesize overviews
   for split modules, merge with carried-forward unchanged records.

> The v1 **Map** phase and its Tarjan-SCC / topological-batching machinery have been
> removed. Dependencies are still *noted* per unit as free-text, but no longer drive
> ordering — extraction is a flat dynamic fan-out.

Agents are **read-only by design**; the slash command handles all file rendering.

### Extraction Skill

`.claude/skills/codebase-context-extractor/SKILL.md` governs extraction:

- Golden rule: *"describe what it does, why, and how the user experiences it — never how
  it's styled."* UI behavior/flows are in scope; visual styling is not.
- Never hallucinate; unsupported claims become open questions.
- References by file path or `path::symbol` — **no line numbers**.
- Scoped to the unit's globs only (no overlap, prevents duplicate output).
- Returns structured JSON (not markdown) for the command to render.

## Known limitations

1. **No cascading staleness.** A changed dependency does not auto-invalidate its dependents
   (bounds re-run cost). Re-run with `forceRefresh` if you need a full pass.
2. **No AST-accurate dependency graph.** Dependencies are noted heuristically as names only.
3. **No hallucination verifier agent.** Strict prompting only — periodic manual spot audits
   of the `keyReferences` are recommended.
4. **NL parsing is best-effort.** The command echoes the resolved plan so a misparse is
   visible before extraction runs.

## Troubleshooting

**"Recon phase failed"** — Ensure this is a git repo (`git status` works) and Claude Code can
read the source files.

**"No modules identified"** — Add module-boundary guidance to `docs/00-overview.md` and re-run.

**Modules keep getting re-extracted** — Check `index.md`'s `last_run_commit` is a valid commit.
If git diff misbehaves, delete `index.md` and re-run for a fresh pass.

**"Removed" section appears** — Source for a previously-documented module is gone. Delete the
doc folder or restore the source, then re-run.

## License

A generic, open-ended tool for codebase documentation. Use it freely on any codebase.

---

> Built as R&D at Brain Station 23 PLC, exploring how existing codebases can be brought to
> AI-DLC readiness — durable, machine-readable context that lets AI coding assistants work on
> a codebase without re-deriving it from scratch each time.
