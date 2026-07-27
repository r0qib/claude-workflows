# Claude Workflows & Skills Repository

A collection of Cloud Code workflow skills and utilities organized by folder.

## Structure

Each folder contains a complete skill or workflow package with its own documentation:

- **document-codebase/** — Workflow and skill for generating comprehensive codebase documentation using Claude
  - Includes workflow, skill, and command integrations
  - See [document-codebase/README.md](document-codebase/README.md) for details

- **justdoit/** — Slash command that runs a feature end-to-end (brainstorm → design → implement → test, commit opt-in) from one natural-language prompt, interactive or autonomous
  - Command + two skills (commit-batcher, assumption-journal); channels the `/sc:*` behavioral modes
  - See [justdoit/README.md](justdoit/README.md) for details

## Getting Started

1. Browse individual folder READMEs for setup and usage instructions
2. Each package includes its own `.claude` directory with workflows, skills, and commands
3. Installation instructions are provided in each package's `INSTALL.md`

## About This Repository

This repository stores reusable Cloud Code artifacts including:
- **Workflows** — Multi-step orchestrations defined in `.claude/workflows/`
- **Skills** — Custom commands and processes in `.claude/skills/`
- **Commands** — CLI integrations in `.claude/commands/`

Explore individual packages for how to integrate them into your Claude Code environment.
