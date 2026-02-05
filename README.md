# Runpod Agent Skills

Skills for AI agents to manage GPU workloads on Runpod.

## Available Skills

### runpodctl

Manage GPU pods, serverless endpoints, templates, volumes, and models.

## Installation

```bash
npx skills add runpod/skills
```

Works with Claude Code, Cursor, GitHub Copilot, Windsurf, Cline, and [17+ other AI agents](https://skills.sh/).

## Setup

```bash
runpodctl doctor
```

## Usage

Ask your AI agent:

- "Create a pod with an RTX 4090"
- "List my pods"
- "What GPUs are available?"
- "Show my account balance"
- "Deploy a serverless endpoint"

## URLs

### Pod URLs

Access exposed ports on your pod:

```
https://<pod-id>-<port>.proxy.runpod.net
```

Example: `https://abc123xyz-8888.proxy.runpod.net`

### Serverless URLs

```
https://api.runpod.ai/v2/<endpoint-id>/run        # Async request
https://api.runpod.ai/v2/<endpoint-id>/runsync    # Sync request
https://api.runpod.ai/v2/<endpoint-id>/health     # Health check
https://api.runpod.ai/v2/<endpoint-id>/status/<job-id>  # Job status
```

## Structure

```
runpodctl/
└── SKILL.md
```

## License

Apache-2.0
