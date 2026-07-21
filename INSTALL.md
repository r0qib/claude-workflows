# Installation

Copy `document-codebase` into your target repository in three steps.

## Step 1: Copy the skill

```bash
cp -r skills/codebase-context-extractor <your-repo>/.claude/skills/
```

## Step 2: Copy the workflow

```bash
cp workflows/document-codebase.js <your-repo>/.claude/workflows/
```

## Step 3: Copy the slash command

```bash
cp commands/document-codebase.md <your-repo>/.claude/commands/
```

## Done

In your target repository, you can now run:

```
/document-codebase
```

The workflow will auto-detect your codebase's stack and structure, then extract documentation into `docs/ai-context/`.

See `README.md` for usage details.
