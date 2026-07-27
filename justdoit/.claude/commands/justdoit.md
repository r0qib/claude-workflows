---
description: Run a feature end-to-end — brainstorm → design → implement → test — in one interactive or autonomous pass, channeling your /sc behavioral modes. Commit is opt-in.
argument-hint: "[plain English — the feature to build + how to run it, e.g. 'add password reset, ask me if unsure' or 'build the export button, I'm away, use your judgment']"
---

# /justdoit

Drive your whole feature workflow from a single prompt. `/justdoit` runs the same
chain you run by hand — **brainstorm → design → implement → test** — and stops with
your changes ready to review. It channels the behavioral modes of your `/sc:*`
commands (brainstorm, design, implement, test, git) **in-session**, gating each phase
the way you would manually.

Two ways to run it, both chosen in plain language inside the prompt:

- **Interactive (default)** — pauses at real decision points and asks you.
- **Autonomous** — "use your best judgment, I'm away" → runs through, logging every
  assumption so you can audit it on return.

**Commit is opt-in.** By default the run ends after `test` with a **dirty working
tree** — nothing is committed. Commits happen only if your prompt explicitly asks for
them ("...and commit it", "commit in batches when done"). Pushing never happens unless
you ask.

## How it works

**Channel, don't call.** The `/sc:*` files are behavioral *prompts*, not functions.
Printing `/sc:brainstorm` does nothing (slash commands fire only from *your* input).
So each phase below INLINES a distilled behavioral contract and this command adopts it
in-session. This command is the single authority on gate behavior — it does **not**
re-read the `/sc/*.md` files at runtime (that would re-arm the hard stops it overrides).

**Five phases, five gates.** The canonical order is `brainstorm → design → implement →
test → commit`. Each phase boundary is a gate. Interactive mode HONORS the stop (pauses
and asks). Autonomous mode OVERRIDES it with an explicit bracketed directive and
continues, logging the assumption.

**Phase toggles in natural language.** "just brainstorm and design", "skip testing",
"no commits" trim the phase set. The resolved plan is echoed in one line before any work
runs, so a misparse is visible immediately.

**Non-destructive.** Never `git add -A`, never push/force/reset/amend, never bypass
hooks, never fold your pre-existing uncommitted work into a pipeline change. See the
Safety Tiers below.

## Usage

```
/justdoit [plain-English feature + how to run it]
```

Examples:

- `/justdoit add a CSV export button to the reports page` — interactive, full chain
  through test, stops dirty.
- `/justdoit build password reset, ask me if you have questions, I'm here` — interactive.
- `/justdoit add rate limiting to the API, use your best judgment, I'm away` — autonomous,
  stops dirty.
- `/justdoit just brainstorm and design a notifications system` — stops after design.
- `/justdoit implement and test the design in docs/plan.md, I'm away` — skips brainstorm
  + design (needs that spec to exist).
- `/justdoit add the health endpoint and commit it in batches when done` — includes the
  commit phase (opt-in).
- `/justdoit refactor the auth guard, skip testing` — drops the test phase (flagged).

## Step 1 — Parse the instruction into a control object (natural language)

Read `$ARGUMENTS` holistically. Classify each clause as **CONTROL** (a meta-instruction
about *how to run the pipeline*) or **DESCRIPTION** (what to build). Concatenate the
DESCRIPTION clauses, in order, into `featureDescription`. Derive the control object below;
when the text is silent on a field, use the default and record it in `assumptions[]`.

**Semantic guard (highest-risk misparse):** a lexicon word is CONTROL only when it is an
imperative aimed at you, not when it is the object of what the software does. In
*"build a UI that lets users skip tests"*, "skip tests" is DESCRIPTION — the feature —
not a phase toggle. When unsure, treat it as description and let the echo catch it.

| Arg | Type | Default | Derive from phrases like… |
|---|---|---|---|
| `featureDescription` | string | — (required) | the residual text after control clauses are stripped |
| `title` / `featureSlug` | string | derived | a short title + kebab-case slug from the description |
| `mode` | `interactive` \| `autonomous` | `interactive` | autonomous ONLY on a clear cue: "use your best judgment", "don't ask me", "I'm away/afk", "full auto", "handle it end to end". Interactive cue: "ask me if…", "I'm here", "pause at decisions", "check with me" |
| `runPhases` | ordered subset of `[brainstorm,design,implement,test,commit]` | `[brainstorm,design,implement,test]` | see toggles below. **`commit` is NOT in the default** — it is added only by an explicit commit instruction |
| `brainstorm.depth` | `shallow`\|`normal`\|`deep` | `normal` | "think hard/go deep"→deep; "quick/sketch/lightweight"→shallow |
| `brainstorm.strategy` | `systematic`\|`agile`\|`enterprise` | `systematic` | the named word |
| `design.type` | `architecture`\|`api`\|`component`\|`database`\|null | `null` (auto) | the named word |
| `design.format` | `spec`\|`diagram` | `spec` | "with a diagram"→diagram. **`code` is forbidden** — downgrade to spec and log it |
| `implement.framework` | string\|null | `null` | "in react/vue/express/…" |
| `implement.safe` | bool | `false` | "be careful", "safe mode", "conservative" |
| `implement.generateTests` | bool | `true` | "don't write tests"→false. Default true because the test phase RUNS tests, it does not author them |
| `test.type` | `unit`\|`integration`\|`e2e`\|`all` | `unit` | "e2e/browser"→e2e, "integration"→integration, "everything"→all. integration/e2e also need the safety opt-in below |
| `test.coverage` | bool | `false` | "with coverage" |
| `commit.push` | bool | `false` (always unless asked) | only "push it", "open a PR" |
| `commit.branch` | string\|null | `null` (slug) | "on a branch called X" |
| `onTestFailure` | `stop` \| `commit-passing` \| `commit-wip` | `stop` | "commit whatever passes"→commit-passing; "commit as WIP"→commit-wip. Default `stop` = leave the tree as-is |
| `artifacts.write` | bool | `false` interactive · `true` autonomous | "write it down/put docs in /docs"→true; "keep it in chat"→false |
| `holdAtDesignGate` | bool | `false` | "hold at the design gate if unclear", "don't implement on guesses" → true (autonomous only) |

**Phase-set toggles → `runPhases`:**

| Phrase | Effect |
|---|---|
| (silent on scope) | `[brainstorm, design, implement, test]` (stops dirty, no commit) |
| "just brainstorm and design", "stop after design", "don't implement yet", "no coding" | end at `design` |
| "just brainstorm", "only requirements" | end at `brainstorm` |
| "skip testing", "no tests" | drop `test` (⚠️ flag: changes become compile-verified only) |
| "skip brainstorm, requirements are ready", "already designed, just implement and test" | move the start later — REQUIRES the prerequisite artifact to exist (see below) |
| "…and commit it", "commit when done", "commit in batches" | ADD `commit` to the end |
| "no commits", "don't commit", "leave it uncommitted" | ensure `commit` is absent (it already is by default) |
| "push it", "open a PR" | ADD `commit` AND set `commit.push = true` |
| "from scratch", "start over" | ignore any prior artifacts, start at `brainstorm` |

**Ambiguity resolution:**
- Silent on mode → `interactive`. Both an autonomous and an interactive cue present →
  `interactive` wins (more restrictive); record it in `conflicts[]`.
- Hedged/hybrid mode ("handle the easy parts, ping me on the big stuff") → `interactive`,
  logged. If a prompt still resolves to autonomous with *low confidence*, the
  design→implement gate BLOCKS once before crossing even in autonomous mode.
- "just/only/up to `<phase>`" = a contiguous run THROUGH that phase from the natural
  start, never that phase alone. Moving the start later requires an explicit
  "already done / ready" cue AND the prerequisite artifact.
- Empty `featureDescription` after stripping: interactive → ask "what should I build?"
  before starting; autonomous → STOP loudly, never invent a feature.

## Step 2 — Echo the resolved plan (one line, before any work)

Print exactly one line so a misparse is visible, then the assumptions. Example:

```
Feature: CSV export button
Resolved: mode = interactive · run = brainstorm → design → implement → test · brainstorm = normal/systematic · tests = unit · on-fail = stop (tree left dirty) · commit = NO (not requested) · push = never · artifacts = in chat
Assumed (input was silent): interactive mode; no commit; unit tests only.
```

Render only the phases that will actually run, so an over-truncation is obvious at a
glance. Then:
- **interactive** → proceed to the first phase; the gates will pause.
- **autonomous** → proceed straight through; do NOT block on the echo (except the
  low-confidence design→implement exception above). Every gate crossing is logged.

## The run state (single source of truth)

Track the run in a compact header you reprint at the START of every phase and every
checkpoint:

```
[justdoit] slug=<slug> · mode=<mode> · phases=<a→b→c> · current=<phase> · on-fail=<policy> · next-gate=<X→Y (honor|override)>
```

Re-derive gate behavior from *this header's mode every time*, never from memory.
**If you cannot state the current phase and mode, stop and re-establish them before
acting.** In autonomous mode (or when `artifacts.write=true`), persist this to
`docs/justdoit/<slug>/state.json` plus an append-only `journal.md` so a walked-away or
interrupted run is auditable and resumable (see the assumption-journal skill). In
interactive mode with `artifacts.write=false`, the conversation itself is the record.

## Per-phase behavioral blocks

Each block: adopt the distilled contract, obey the **path allow-list**, produce the
artifact, then handle the gate per mode. The allow-list is enforced by legality, not by
hope — a write outside it is a gate violation: stop and flag.

### Phase: BRAINSTORM
- **Goal:** Socratic requirements discovery → a requirements brief (≥1 functional
  requirement + ≥1 acceptance criterion). No design, no code.
- **Path allow-list:** may write ONLY the requirements brief (to `docs/justdoit/<slug>/requirements.md`
  when `artifacts.write`, else hold it in the conversation). No `src/` writes.
- **MCP:** sequential + context7 + serena. Never magic/morphllm.
- **Interactive:** run the real Socratic Q&A *in the foreground* — do NOT delegate the
  dialogue to subagents (they cannot ask you). Present the brief; honor the stop.
- **Autonomous:** self-answer each question with best judgment under **prompt-anchored
  scope** — tag every requirement PROMPT-STATED vs AGENT-PROPOSED; only PROMPT-STATED
  (plus strict technical prerequisites) enter the build set. AGENT-PROPOSED are recorded
  as "suggested, NOT built". Log each assumption. Cross with:
  `[gate brainstorm→design | autonomous] Requirements captured (<path/summary>). The driving prompt authorized the full chain; continuing to design. Assumptions: A-1..A-k.`

### Phase: DESIGN
- **Goal:** consume the requirements brief (Read it fresh, do not rely on memory).
  Produce a design: architecture, API contracts, data models, interfaces, AND a required
  **acceptance-criteria → test map** (a checklist with stable criterion IDs). This map is
  the contract the test phase verifies against.
- **Path allow-list:** may write ONLY `design.md` (or hold in conversation). May contain
  interface signatures / pseudocode — **never full function bodies, never source files,
  never `src/` writes.** `design.format=code` is forbidden. Any `src/` write here = gate
  violation.
- **MCP:** sequential + context7 + serena.
- **Interactive — the strict gate.** Present a short, scannable *key-decisions + risks*
  summary (not the whole doc) and ask open-ended: "approve & implement, revise, or stop?".
  **Fail closed:** cross only on an explicit, unambiguous affirmative. Any hedge, embedded
  question, or new constraint = REVISE and re-pose the gate. Never infer approval from
  non-committal prose. On revise, version the artifact (`design.v1`, `v2`, …) so a prior
  version is recoverable.
- **Autonomous.** Before crossing, run a **stateless read-only review subagent** (fresh
  context) that reads the requirements + design and returns {covers-all-criteria?,
  blocking-risks, security/data-model red-flags}. It only returns a verdict — it never
  pauses — so it does not break the foreground constraint. Then:
  - If it flags a **blocking** architectural/technology choice with cross-slice blast
    radius (transport, persistence, auth model, sync/async, layering), or any high-impact
    irreversible item (schema/data migration, public API shape, security posture, data
    deletion) → **DEFER that slice**, build the unambiguous ones, surface it as "needs
    human decision". Do not self-approve it.
  - Else cross with:
    `[gate design→implement | autonomous] Design reviewed (covers the criteria, no blocking risks). Proceeding to implement. Design at <path/summary>.`
  - If `holdAtDesignGate` is set, OR mode-confidence was low → halt here and report
    instead of crossing.
- **State 1 (implement not in `runPhases`):** never cross — this is how "just brainstorm
  and design" terminates. Report and finalize.

### Phase: IMPLEMENT
- **Goal:** write the feature code AND its tests (test *authoring* lives here, because the
  test phase only runs tests). Follow the design; stay within prompt-anchored scope.
- **Path allow-list:** the ONLY phase that may Write/Edit under the repo's source roots
  (`src/`, etc.) and test dirs. Snapshot each test file's hash at authoring time.
- **MCP:** context7 + sequential + playwright. Never magic. For UI-primary work with no
  existing component conventions to mine: in autonomous mode either DEFER the UI slice as
  "needs human review" or require an explicit "hand-roll the UI" opt-in; flag UI built
  without magic as a top-of-report HIGH-RISK item, not a footnote.
- **Gate (handoff, not a stop).** Interactive: light checkpoint, default-yes ("implementation
  compiles — run tests?"). Autonomous:
  `[gate implement→test | autonomous] Implementation done-marker is a handoff. Continuing to test.`
- **In-phase escalation (interactive):** a mid-phase decision matching a blocking trigger
  (security posture, persisted data model, public API shape, irreversible/destructive op,
  adding a dependency, auth/permission logic, architecture with cross-slice blast radius)
  MUST interrupt and ask immediately — do not defer it to the next checkpoint.

### Phase: TEST
- **Goal:** RUN + analyze only. Never author tests here (that is implement's job).
- **Opens with a coverage-gap check:** map each design criterion ID to a named test. Any
  criterion with no test → the check FAILS the phase → re-enter implement-mode to author
  the missing test, then run. If no criteria map exists (e.g. a start-later run against a
  free-prose spec): autonomous re-enters design-mode to synthesize the map (logged as a
  loud REVISIT); interactive asks.
- **Bounded fix loop — max 3 attempts per slice.** Each cycle: analyze → classify
  {code-bug | env/missing-service | flaky} → re-enter implement-mode to fix **only
  non-test source** → re-run.
  - **Test files are immutable during the loop.** The path of least resistance —
    weakening an assertion to go green — is forbidden. If you believe a *test* is wrong
    (not the code), that slice is BLOCKED for human review, logged with the criterion ID
    it contradicts; it is not auto-fixed.
  - **env/missing-service failures = BLOCKED**, reported as an environment gap. Do not
    burn attempts; never re-run a known side-effecting command in the loop.
  - **Cycle detection:** track the set of error signatures seen; bail if one recurs or
    after 2 stalled iterations, rather than always burning all 3.
- **After the loop:** apply `onTestFailure`. Default `stop` → leave the tree exactly as-is,
  write the report, do not proceed to commit. (`commit-passing` / `commit-wip` only if the
  prompt asked for that policy.)
- **`skip testing`** removes this phase and test authoring; the echo must state that the
  result is compile-verified only.

### Phase: COMMIT (opt-in — only runs if in `runPhases`)
- Follow the **commit-batcher** skill. Group changed files into several meaningful
  conventional commits (never one blob); stage each group by **explicit path**
  (`git add <paths>` — never `-A`/`.`); keep each slice's tests with its code; order
  dependency-first; borrow message *style* from `/sc:git` but the orchestrator runs all
  git itself (never channel `/sc:git` as an executor).
- **Content secret scan** on each staged diff (high-entropy strings, `sk_`/`AKIA`/`ghp_`,
  private-key headers) → block + report if it fires. Never commit `.env`/`*.pem`/creds.
  Never include `docs/justdoit/` run artifacts in a code commit.
- Interactive: present the commit plan for approval (commit / regroup / hold) before
  committing. Autonomous: apply and log the plan.
- **Push is a universal hard gate** — never push unless `commit.push` was explicitly set;
  never force-push, never push to main unasked. The run otherwise ends at "committed
  locally".

## Safety Tiers (apply in every mode; autonomy does NOT relax them)

**Tier 1 — always forbidden** (autonomy is not the confirmation these need): any remote
mutation (push, force-push, publish, deploy, mutating external APIs); destructive/history
git (`reset --hard`, `checkout -- .`/`restore .`, `clean -fd`, `branch -D`, shared-history
rebase, amending existing commits — always NEW commits); hook/signature bypass
(`--no-verify`, `--no-gpg-sign`); `git config` changes / touching `.git/` internals;
`git add -A`/`.`; deleting or overwriting pre-existing files the feature didn't create;
committing secret-bearing files; dependency downgrades / new runtime deps / major bumps
not named in the design; migrations against a non-local DB; edits to shared infra
(CI, Dockerfile, deploy/flag config) unless the design explicitly scoped them.

**Tier 2 — requires an explicit NL opt-in:** remote push, adding dependencies, committing
to main/master directly, editing infra config, hand-rolling UI without magic, running
integration/e2e (external-service) tests.

**Tier 3 — allowed unattended:** create/edit source in feature scope; create/use a
feature branch; run build/test/format/lint (external integrations OFF by default —
unit scope first; install with scripts disabled unless opted in); multiple meaningful
LOCAL commits *only if the commit phase was requested*.

**Pre-flight (before any write):** record the baseline {branch, HEAD SHA, dirty?}.
**Dirty-tree precondition:** if the tree is dirty at the start of an autonomous run,
either auto-stash the user's changes and report the stash ref, or refuse to start
("working tree dirty — commit or stash first"). NEVER stage a path that already has
un-staged user modifications. If a commit phase is requested and you are on main/master,
create + checkout a feature branch first. Only ever suggest `git reset --hard <baseline>`
as a rollback when the baseline was clean; otherwise give a branch-delete recipe that
cannot touch pre-existing WIP. The orchestrator never runs a destructive rollback itself.

## Interrupts & resume

- **Mid-phase user message (interactive):** never read it as approval or as a stop.
  Classify {clarification | in-phase steer | upstream-invalidating}. Answer first, then:
  clarification/steer → apply and re-pose the original gate unchanged; upstream-invalidating
  (e.g. "use Postgres not Mongo" mid-implement) → checkpoint, roll back to the owning phase,
  re-version its artifact, replay forward. A dismissed AskUserQuestion is an interrupt, not
  an answer.
- **Resume (when artifacts were written):** on re-invocation with a non-done `state.json`,
  autonomous auto-resumes after a staleness check; interactive offers to resume. Re-run
  only pending/failed/deferred work; before re-running an in-flight slice, reset ONLY that
  slice's uncommitted paths to baseline (never a global reset).

## Final report

Close every run with:
- resolved plan + phases actually run;
- artifact locations (or "held in conversation");
- what was built vs deferred vs blocked, with reasons;
- test results; any slice still failing after 3 attempts + why;
- in autonomous mode: every assumption (with confidence + reversibility), any REVISIT
  items, any Tier-1/Tier-2 blocks that fired ("wanted lodash — not authorized, used native"),
  UI-without-magic HIGH-RISK notes;
- the git state: **"working tree left dirty — nothing committed"** unless a commit phase
  ran, in which case list commits (SHA + message) and "push withheld" unless pushed;
- exact resume / rollback instructions when artifacts were written.

## Troubleshooting

**"It started coding during design"** — a `src/` write in the design phase is a gate
violation; the allow-list should stop it. Report it as a bug in the run.

**"It committed without being asked"** — it shouldn't: `commit` is not in the default
phase set. Only an explicit commit instruction adds it. Check the Step-2 echo — it states
`commit = NO` unless requested.

**"Autonomous run stalled after brainstorm"** — the override directive at the
brainstorm→design gate is what continues past the channeled stop; if it stalled, the
directive wasn't emitted. This command, not the `/sc` files, owns gate behavior.

**"The command does nothing / channels undefined behavior"** — the SuperClaude `/sc:*`
commands (brainstorm, design, implement, test, git) must be installed in
`~/.claude/commands/sc/`. See INSTALL.md.

**"It touched my uncommitted work"** — it must not. Autonomous runs stash-or-refuse on a
dirty tree and never stage a path with pre-existing edits. Report this as a bug.
