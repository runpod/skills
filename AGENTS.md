# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## Overview

This is a **skills repository and plugin marketplace** for AI agents (Claude Code, Cursor, Copilot, etc.) to manage GPU workloads on Runpod. It contains no application code — only skill definition files (`SKILL.md`) and plugin manifests that teach AI agents how to use the Runpod CLIs.

Skills are installed by users via `npx skills add runpod/skills` (see [skills.sh](https://skills.sh/)) or as a Claude Code plugin marketplace via `/plugin marketplace add runpod/skills`.

## Repository Structure

The repository is both a Claude Code plugin marketplace (via `.claude-plugin/marketplace.json`) and an Agent Skills repository. Each plugin lives in its own directory:

```
.claude-plugin/marketplace.json          — marketplace catalog
runpodctl/.claude-plugin/plugin.json     — runpodctl plugin manifest
runpodctl/skills/runpodctl/SKILL.md      — runpodctl skill definition
flash/.claude-plugin/plugin.json         — flash plugin manifest
flash/skills/flash/SKILL.md             — flash skill definition
```

## Skill File Format

`SKILL.md` files use YAML frontmatter with these fields:
- `name`, `description` — skill identity
- `allowed-tools` — tool permissions (e.g., `Bash(runpodctl:*)`)
- `compatibility` — supported platforms
- `metadata` — author, version
- `license`

The body is markdown documentation that agents consume to learn the CLI commands.

## Plugin Manifest Format

Each plugin has a `.claude-plugin/plugin.json` with:
- `name`, `description` — plugin identity
- `version` — semantic version for update tracking
- `author` — plugin author
- `license`

## Conventions

- **Spelling:** "Runpod" (capital R). The CLI command is `runpodctl` (lowercase).
- **License:** Apache-2.0
- Keep plugins sorted alphabetically in marketplace.json and the README.md table.
