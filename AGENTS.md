# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## Overview

This is a **skills repository** for AI agents (Claude Code, Cursor, Copilot, etc.) to manage GPU workloads on Runpod. It contains no application code — only skill definition files (`SKILL.md`) that teach AI agents how to use the `runpodctl` CLI.

Skills are installed by users via `npx skills add runpod/skills` (see [skills.sh](https://skills.sh/)).

## Repository Structure

Each skill lives in its own directory with a `SKILL.md` file containing frontmatter metadata and CLI documentation:

```
runpodctl/SKILL.md   — runpodctl CLI skill definition
```

## Skill File Format

`SKILL.md` files use YAML frontmatter with these fields:
- `name`, `description` — skill identity
- `allowed-tools` — tool permissions (e.g., `Bash(runpodctl:*)`)
- `compatibility` — supported platforms
- `metadata` — author, version
- `license`

The body is markdown documentation that agents consume to learn the CLI commands.

## Conventions

- **Spelling:** "Runpod" (capital R). The CLI command is `runpodctl` (lowercase).
- **License:** Apache-2.0
