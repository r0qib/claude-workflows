---
name: assumption-journal
description: For unattended/autonomous orchestrations — turn every point that WOULD have been a question into a logged, auditable decision, and persist resume state. Use when running a multi-phase pipeline without a human at the keyboard. Conversation-first; writes files only when asked or when running autonomously.
---

# assumption-journal

When a pipeline runs without a human to ask, every judgment call that *would* have been an
interactive question must instead become a **logged assumption** — so the returning user
can audit exactly what was decided and why. This skill defines that journal, the resume
state, and the re-grounding rule.

## When to use

- Any autonomous / unattended run of a multi-phase orchestrator (e.g. `/justdoit` in
  autonomous mode).
- Any long interactive run where the user asked for a written record ("write it down",
  "put the docs in /docs").

**Conversation-first.** In interactive mode with no request to write files, the
conversation itself is the record — do NOT create files. Write the journal and state only
when the run is autonomous OR the user explicitly asked for durable artifacts.

## Where it lives

When writing is enabled, use a per-run folder: `docs/justdoit/<slug>/` containing
`journal.md` (append-only, human-readable), `state.json` (machine-readable resume state),
and the phase artifacts (`requirements.md`, `design.md`, versioned `design.v1.md`…).
Keep this folder out of code commits; the user can gitignore it.

## journal.md

Start with a header, then append one entry per judgment call. Never rewrite past entries.

```
# justdoit run journal — <title>
run: <slug> · started: <ts> · mode: <mode> · baseline: <branch>@<HEAD SHA>
resolved plan: <the one-line echo>

## <ts> · <phase> · <type>
what:        <one line — the decision/assumption/action>
why:         <the reasoning>
alternatives: <what else was considered> (for decisions)
confidence:  high | medium | low
reversibility: reversible | hard | irreversible
revisit?:    <yes + what-would-change-the-answer, or no>
artifacts:   <files touched / commit SHA if any>
```

`type` ∈ `assumption` | `decision` | `action` | `failure` | `deferral` | `safety-block`.

Log, at minimum: every default taken from a silent/ambiguous prompt; every gate crossed
autonomously; every PROMPT-STATED-vs-AGENT-PROPOSED scope call; every deferred slice and
why; every fix-loop failure + root cause; every Tier-1/Tier-2 safety block that fired
("wanted to add lodash — not authorized, used native"); any test file modified after
authoring.

## state.json (resume)

```json
{
  "runId": "<slug>",
  "mode": "autonomous",
  "resolvedPlan": { "...": "the parsed control object" },
  "baseline": { "branch": "...", "head": "<sha>", "dirty": false },
  "phasesDone": ["brainstorm", "design"],
  "currentPhase": "implement",
  "slices": [
    { "id": "list-endpoint", "status": "committed", "paths": ["..."], "commitSha": "...", "testHash": "..." },
    { "id": "create-endpoint", "status": "failed", "paths": ["..."], "reason": "..." }
  ],
  "pendingDecision": null,
  "lastCheckpoint": "<ts>"
}
```

`slices[].status` ∈ `pending` | `implemented` | `tested` | `committed` | `failed` |
`deferred`. Persist a `pendingDecision` (phase, question, options, tag, ts) the instant a
gate question is posed; clear it only when a real answer is recorded — so an interrupted
run never treats an unanswered gate as passed.

## Re-grounding rule (defeats context drift)

`state.json` is the single source of truth for a written run. At the START of every phase
and every checkpoint, re-read it and reprint the compact run header (runId, mode,
phases, currentPhase, on-fail policy, next gate + honor/override). Re-derive gate behavior
from the freshly-read mode every time, never from memory. **If you cannot cite the current
phase and mode, stop and re-read before acting.**

## Resume

On re-invocation with a non-`done` state.json: verify each `committed` slice's SHA still
exists (idempotent skip); re-run only `pending`/`failed`/`deferred` slices. Before
re-running an in-flight slice, reset ONLY that slice's uncommitted paths to baseline —
never a global reset. If `pendingDecision` is non-null, replay it before doing anything
else. Autonomous auto-resumes after a staleness check; interactive offers to resume.

## Final report

Surface the journal in the closing report: enumerate every REVISIT assumption, every
deferred/failed/blocked slice with reasons, and render assumption entries INLINE next to
any commit they relate to — so "green" or "done" never reads as "correct" without its
caveats attached.
