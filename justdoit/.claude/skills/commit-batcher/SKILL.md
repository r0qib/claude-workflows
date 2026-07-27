---
name: commit-batcher
description: Group a changeset into several meaningful conventional commits (never one blob), staging by explicit path only. Use whenever committing multi-file work — especially after a /justdoit build. Never git add -A, never push or amend unless explicitly asked, always non-destructive.
---

# commit-batcher

A discipline for turning a pile of changes into a clean, reviewable, bisectable commit
history — several meaningful commits that each group related files, rather than one big
blob. Commit only when the user has explicitly asked; otherwise leave the tree dirty.

## When to use

- After a feature build (e.g. the `/justdoit` commit phase) when the user asked to commit.
- Any time you are about to commit multi-file work and want coherent, related-files commits.

Do NOT invoke this to commit speculatively. If the user did not ask for a commit, stop —
leave the working tree as-is.

## Core rules

1. **Never one blob.** Partition the changeset into groups and make one commit per group.
   A single catch-all commit is the anti-pattern this skill exists to prevent.

2. **Group by slice first, then by layer.** Partition by feature/design-slice boundary
   first. Within a large slice, split by layer in dependency order:
   schema/migration → backend/service → API/route → frontend → docs. Order the commits so
   each builds on the last (dependency-first).

3. **Keep tests with the code they verify.** A slice's tests go in the SAME commit as the
   code, so every commit is self-consistent and ideally green in isolation. Only fall back
   to a trailing per-slice test commit when the test↔code mapping is genuinely ambiguous —
   and mine the repo's existing test layout before deciding.

4. **Stage by explicit path — NEVER `git add -A` / `git add .`.** Stage exactly the files
   in the current group: `git add <path1> <path2> …`. Explicit staging is what makes
   per-slice isolation possible and prevents sweeping in secrets, scratch files, or a
   held-back failing slice.

5. **Conventional messages, orchestrator-run.** Borrow the conventional-commit style
   (`feat:`, `fix:`, `test:`, `refactor:`, `docs:`, `chore:`) for the message wording.
   Run git yourself with `git commit -F <msgfile>` (or `-m`). Do not delegate git
   execution to another command. Always create NEW commits — never `--amend`.

6. **Secret scan before every commit.** Scan the exact staged diff for high-entropy
   strings and known key shapes (`sk_`, `AKIA`, `ghp_`, `-----BEGIN … PRIVATE KEY-----`).
   If anything matches, do NOT commit — unstage and report. Never commit `.env`, `*.pem`,
   `credentials*`, or similar. Never commit pipeline run-artifacts (e.g. a
   `docs/justdoit/<slug>/` folder) inside a code commit.

7. **Never push, never rewrite history, unless explicitly asked.** No `push`, `push
   --force`, `reset --hard`, `rebase` of shared history, `branch -D`, or hook/signature
   bypass (`--no-verify`, `--no-gpg-sign`). Pushing requires a distinct, explicit request
   and is never implied by "commit".

8. **Hooks are signal, not obstacle.** If a pre-commit hook fails, fix the underlying
   issue (e.g. run the formatter it wants), re-stage, and make a NEW commit. Never bypass.

## Procedure

1. `git status` + `git diff` to see the full changeset. Confirm the user asked to commit.
2. Partition files into ordered groups (rules 2–3). Draft a one-line conventional message
   per group focused on the *why*.
3. Present the plan (group → files → message) for approval when interactive; log it when
   autonomous.
4. For each group in dependency order: `git add <explicit paths>` → secret-scan the staged
   diff → `git commit -F <msg>`. If a hook fails, fix, re-stage, new commit.
5. Report the commits made (SHA + message). State "push withheld" unless a push was
   explicitly requested.

## Handling a held-back failing slice

If some slices pass and one fails (e.g. after a bounded fix loop), the default is to
**commit nothing and leave the tree dirty** unless the user chose a "commit what passes"
policy. If they did: verify each green slice builds in isolation before committing it —
if a green slice shares a file with the failing slice and cannot be cleanly staged apart,
hold it too rather than emit a non-bisectable "green" commit.
