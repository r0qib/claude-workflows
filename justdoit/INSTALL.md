# Installation

Copy `justdoit` into your target repository. Two skills + one command, no workflow.

## Prerequisite — the SuperClaude `/sc:*` commands

`justdoit` channels the behavioral modes of these SuperClaude commands, so they must
already be installed in your environment (`~/.claude/commands/sc/`):

- `/sc:brainstorm`, `/sc:design`, `/sc:implement`, `/sc:test`, `/sc:git`

The orchestrator does not call or read these files at runtime — it inlines a distilled
contract for each phase — but they define the workflow vocabulary this package assumes.
Without them installed the command still runs, but you lose the shared behavioral
grounding; install them first. (Get them from your SuperClaude setup.)

Recommended MCP servers: `sequential`, `context7`, `playwright`, `serena`. The command
never requires `magic` or `morphllm`.

## Step 1: Copy the skills

```bash
cp -r justdoit/.claude/skills/commit-batcher      <your-repo>/.claude/skills/
cp -r justdoit/.claude/skills/assumption-journal  <your-repo>/.claude/skills/
```

## Step 2: Copy the command

```bash
cp justdoit/.claude/commands/justdoit.md <your-repo>/.claude/commands/
```

## Step 3: (recommended) gitignore the run folder

Autonomous runs (and any run where you ask for written docs) create
`docs/justdoit/<slug>/` with the requirements brief, design, journal, and resume state.
It is process metadata, not part of your feature history:

```bash
echo "docs/justdoit/" >> <your-repo>/.gitignore
```

## Done

In your target repository:

```
/justdoit [plain-English feature + how to run it]
```

Examples:

- `/justdoit add a CSV export button` — interactive, full chain through test, stops dirty
- `/justdoit add rate limiting, use your judgment, I'm away` — autonomous, stops dirty
- `/justdoit just brainstorm and design a notifications system` — stops after design
- `/justdoit add the health endpoint and commit it in batches` — includes the commit phase

The command echoes the resolved plan before doing anything, so you can catch a misparse
up front. See `README.md` for the full usage table and design notes.
