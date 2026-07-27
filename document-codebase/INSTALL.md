# Installation

Copy `document-codebase` into your target repository in three steps.

## Step 1: Copy the skill

```bash
cp -r .claude/skills/codebase-context-extractor <your-repo>/.claude/skills/
```

## Step 2: Copy the workflow

```bash
cp .claude/workflows/document-codebase.js <your-repo>/.claude/workflows/
```

## Step 3: Copy the slash command

```bash
cp .claude/commands/document-codebase.md <your-repo>/.claude/commands/
```

## Done

In your target repository, run:

```
/document-codebase [plain-English instructions]
```

Everything is optional and natural-language. Examples:

- `/document-codebase` — document everything, auto agent count
- `/document-codebase use 5 agents, just the auth module` — concurrency + focus
- `/document-codebase re-scan from scratch` — full refresh
- `/document-codebase put docs under .ai` — override the docs root

The workflow scaffolds a fixed folder tree under `docs/` (create-if-missing) and extracts
documentation into `docs/modules/**` and `docs/technical/**`. It never overwrites existing
files. See `README.md` for the full folder tree and usage details.
