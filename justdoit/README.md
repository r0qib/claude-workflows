# justdoit: One-Prompt Feature Pipeline

A reusable Claude Code slash command that runs your whole feature workflow —
**brainstorm → design → implement → test** — from a single natural-language prompt, in
either an interactive or a fully unattended pass. It channels the behavioral modes of your
SuperClaude `/sc:*` commands in-session and gates each phase the way you would by hand.

Commit is opt-in. By default a run ends with your changes ready to review in a dirty
working tree — nothing is committed and nothing is pushed unless you ask.

## What it does

Given one prompt, `/justdoit`:

1. **Parses your intent in plain language** — what to build, and how to run it
   (interactive vs autonomous, which phases, whether to commit) — then **echoes the
   resolved plan** in one line so a misparse is visible before any work starts.
2. **Runs the phases you asked for**, in order: brainstorm → design → implement → test
   (→ commit, only if requested). Each phase adopts the matching `/sc` behavioral mode.
3. **Gates every phase boundary.** Interactive mode pauses and asks you at real decision
   points; autonomous mode continues, logging every assumption it makes.
4. **Stops clean.** Tests run (with a bounded, 3-attempt fix loop); then the run ends with
   the working tree left for your review — unless your prompt explicitly asked to commit.

## The two modes

| You say… | Mode | Behavior |
|---|---|---|
| (nothing about asking) | **interactive** (default) | pauses at the design gate and other real decisions |
| "ask me if you have questions, I'm here" | **interactive** | same, explicitly |
| "use your best judgment, I'm away" / "handle it end to end" | **autonomous** | runs straight through, logs assumptions, defers high-impact/irreversible calls |

## Usage

```
/justdoit [plain-English feature + how to run it]
```

| You say… | Resolves to |
|---|---|
| `/justdoit add a CSV export button` | interactive · brainstorm→design→implement→test · stops dirty |
| `/justdoit build password reset, ask me if unsure` | interactive |
| `/justdoit add rate limiting, use your judgment, I'm away` | autonomous · stops dirty |
| `/justdoit just brainstorm and design a notifications system` | stops after design |
| `/justdoit implement and test the design in docs/plan.md, I'm away` | starts at implement (needs that spec) |
| `/justdoit refactor the auth guard, skip testing` | drops the test phase (flagged) |
| `/justdoit add the health endpoint and commit it in batches` | adds the commit phase (opt-in) |
| `/justdoit ship the export feature and push it` | adds commit + push (both explicit opt-ins) |

## The generated artifacts

By default **nothing is written to disk** — interactive runs keep the requirements brief,
design, and decisions in the conversation. Files appear only when you ask ("write it
down", "put the docs in /docs") or on an **autonomous** run, which needs a durable audit
trail and resume state:

```
docs/justdoit/<slug>/
├── requirements.md     the brainstorm brief
├── design.md           architecture + acceptance-criteria → test map  (design.v1.md… on revise)
├── journal.md          append-only log of every autonomous assumption/decision
└── state.json          resume state (phases done, per-slice status)
```

This folder is process metadata — keep it gitignored; it is never swept into a code commit.

## Design principles

- **Interactive spine.** The pipeline runs in your foreground session so it can pause and
  ask. That is why it ships as a slash command, not a background workflow.
- **Phase-gated.** Five phases, five gates. The design→implement gate is strict and
  fail-closed in interactive mode — a hedge is treated as "revise", never as approval.
- **Channel, don't call.** `/sc:*` files are behavioral prompts, not functions. This
  command inlines each phase's distilled contract and is the single authority on gate
  behavior; it never prints `/sc:x` (inert) and never re-reads the `/sc` files at runtime
  (that would re-arm the stops it deliberately overrides).
- **Commit is opt-in; push is never implied.** The default run stops dirty. Commits happen
  only on an explicit instruction; pushing needs its own explicit ask.
- **Scoped autonomy.** "Don't ask me" authorizes running the *build* unattended — not
  relaxing any safety rail. Destructive git, remote pushes, config/hook bypass, adding
  dependencies, and touching pre-existing uncommitted work stay forbidden or opt-in.
- **Non-destructive.** Explicit-path staging only (never `git add -A`); new commits only
  (never amend); dirty-tree runs stash-or-refuse rather than fold in your WIP.

## How the phases map to your `/sc` commands

| Phase | Channels | Produces | Path it may write |
|---|---|---|---|
| brainstorm | `/sc:brainstorm` | requirements brief | brief only |
| design | `/sc:design` | design + criteria→test map | `design.md` only (no code) |
| implement | `/sc:implement` | feature code **and** tests | source + test dirs |
| test | `/sc:test` (runs only) | test results | — (fix loop re-enters implement) |
| commit | `/sc:git` (message style only) | batched commits | — (git, opt-in) |

Test *authoring* lives in the implement phase, because `/sc:test` runs tests but does not
write them. The test phase opens with a coverage-gap check tying each design criterion to
a named test.

## Requirements

- **Claude Code** with the `AskUserQuestion` capability (for interactive gates).
- **The SuperClaude `/sc:*` commands** installed in `~/.claude/commands/sc/` — this
  command channels their behavioral modes. See `INSTALL.md`.
- **MCP servers** (optional but recommended): `sequential`, `context7`, `playwright`,
  `serena`. The command never depends on `magic` or `morphllm`; UI work falls back to
  framework docs + repo-convention mining and is flagged for extra review.

## Known limitations

1. **Channeling is prompt-driven, not deterministic.** Phase sequencing is steered by
   instructions, not hard control flow. The echoed plan and per-phase re-grounding make
   drift visible, but this is not the guaranteed ordering an SDK program would give.
2. **Autonomous "green" ≠ "correct".** In autonomous mode the code and its tests come from
   the same self-answered assumptions. Low-confidence slices are reported as "verify
   against assumed requirements", not silently trusted.
3. **UI quality is unverified without `magic`.** Behavior is validated (playwright), visual
   correctness is not. UI-primary autonomous work is deferred or flagged HIGH-RISK.
4. **Inert without the `/sc` commands.** If they are not installed, the phase contracts have
   nothing to channel. INSTALL.md calls this out.

## Troubleshooting

**The command does nothing / channels undefined behavior** — install the SuperClaude
`/sc:*` commands in `~/.claude/commands/sc/` (INSTALL.md).

**It started coding during the design phase** — a source write in design is a gate
violation; the per-phase allow-list should block it. Report it.

**It committed without being asked** — it shouldn't: `commit` is not in the default phase
set. Check the Step-2 echo, which states `commit = NO` unless you requested it.

**An autonomous run stalled after brainstorm** — the gate-override directive is what
continues past the channeled stop. This command, not the `/sc` files, owns gate behavior.

## Installation

See `INSTALL.md`. Two skills + one command, no workflow, no dependencies beyond Claude
Code, git, and the `/sc:*` commands.

---

> Built as R&D at Brain Station 23 PLC, exploring how a developer's repeated
> brainstorm→design→implement→test loop can be driven from a single prompt — with real
> human-in-the-loop gates when you're present, and audited, safety-railed autonomy when
> you're away.
