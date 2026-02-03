---
name: runpod
description: Runpod CLI to manage your GPU workloads.
allowed-tools: Bash(runpod:*)
compatibility: Linux, macOS
metadata:
  author: runpod
  version: "2.0"
license: Apache-2.0
---

# Runpod

Manage GPU pods, serverless endpoints, templates, volumes, and models.

> **Spelling:** "Runpod" (capital R). Command is `runpod` (lowercase). Never "RunPod".

## Quick start

```bash
runpod doctor                    # First time setup (API key + SSH)
runpod gpu list                  # See available GPUs
runpod template search pytorch   # Find a template
runpod pod create --template runpod-torch-v21 --gpu-type-ids "NVIDIA RTX 4090"  # Create from template
runpod pod list                  # List your pods
```

API key: https://runpod.io/console/user/settings

## Commands

### Pods

```bash
runpod pod list                                    # List all pods
runpod pod get <pod-id>                            # Get pod details (includes SSH info)
runpod pod create --template runpod-torch-v21 --gpu-type-ids "NVIDIA RTX 4090"  # Create from template (recommended)
runpod pod create --image "runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04" --gpu-type-ids "NVIDIA RTX 4090"  # Create with image
runpod pod start <pod-id>                          # Start stopped pod
runpod pod stop <pod-id>                           # Stop running pod
runpod pod restart <pod-id>                        # Restart pod
runpod pod reset <pod-id>                          # Reset pod
runpod pod update <pod-id> --name "new"            # Update pod
runpod pod delete <pod-id>                         # Delete pod
```

**Create flags:** `--template`, `--image`, `--name`, `--gpu-type-ids`, `--gpu-count`, `--container-disk-in-gb`, `--volume-in-gb`, `--volume-mount-path`, `--ports`, `--env`, `--cloud-type`, `--data-center-ids`

### Serverless (alias: sls)

```bash
runpod serverless list                             # List all endpoints
runpod serverless get <endpoint-id>                # Get endpoint details
runpod serverless create --name "x" --template-id "tpl_abc"  # Create endpoint
runpod serverless update <endpoint-id> --workers-max 5       # Update endpoint
runpod serverless delete <endpoint-id>             # Delete endpoint
```

**Create flags:** `--name`, `--template-id`, `--gpu-type-ids`, `--gpu-count`, `--workers-min`, `--workers-max`, `--data-center-ids`

### Templates (alias: tpl)

```bash
runpod template list                               # Official + community (first 10)
runpod template list --type official               # All official templates
runpod template list --type community              # Community templates (first 10)
runpod template list --type user                   # Your own templates
runpod template list --all                         # Everything including user
runpod template list --limit 50                    # Show 50 templates
runpod template search pytorch                     # Search for "pytorch" templates
runpod template search comfyui --limit 5           # Search, limit to 5 results
runpod template search vllm --type official        # Search only official
runpod template get <template-id>                  # Get template details (includes README, env, ports)
runpod template create --name "x" --image "img"    # Create template
runpod template create --name "x" --image "img" --serverless  # Create serverless template
runpod template update <template-id> --name "new"  # Update template
runpod template delete <template-id>               # Delete template
```

**List flags:** `--type` (official, community, user), `--limit`, `--offset`, `--all`
**Create flags:** `--name`, `--image`, `--container-disk-in-gb`, `--volume-in-gb`, `--volume-mount-path`, `--ports`, `--env`, `--docker-start-cmd`, `--docker-entrypoint`, `--serverless`, `--readme`

### Network Volumes (alias: nv)

```bash
runpod network-volume list                         # List all volumes
runpod network-volume get <volume-id>              # Get volume details
runpod network-volume create --name "x" --size 100 --data-center-id "US-GA-1"  # Create volume
runpod network-volume update <volume-id> --name "new"  # Update volume
runpod network-volume delete <volume-id>           # Delete volume
```

**Create flags:** `--name`, `--size`, `--data-center-id`

### Models

```bash
runpod model list                                  # List your models
runpod model list --all                            # List all models
runpod model list --name "llama"                   # Filter by name
runpod model list --provider "meta"                # Filter by provider
runpod model add --name "my-model" --model-path ./model  # Add model
runpod model remove --name "my-model"              # Remove model
```

### Registry (alias: reg)

```bash
runpod registry list                               # List registry auths
runpod registry get <registry-id>                  # Get registry auth
runpod registry create --name "x" --username "u" --password "p"  # Create registry auth
runpod registry delete <registry-id>               # Delete registry auth
```

### Info

```bash
runpod user                                        # Account info and balance (alias: me)
runpod gpu list                                    # List available GPUs
runpod gpu list --include-unavailable              # Include unavailable GPUs
runpod datacenter list                             # List datacenters (alias: dc)
runpod billing pods                                # Pod billing history
runpod billing serverless                          # Serverless billing history
runpod billing network-volume                      # Volume billing history
```

### SSH

```bash
runpod ssh info <pod-id>                           # Get SSH info (command + key, does not connect)
runpod ssh list-keys                               # List SSH keys
runpod ssh add-key                                 # Add SSH key
```

**Agent note:** `ssh info` returns connection details, not an interactive session. If interactive SSH is not available, execute commands remotely via `ssh user@host "command"`.

### File Transfer

```bash
runpod send <path>                                 # Send files (outputs code)
runpod receive <code>                              # Receive files using code
```

### Utilities

```bash
runpod doctor                                      # Diagnose and fix CLI issues
runpod update                                      # Update CLI
runpod version                                     # Show version
runpod completion bash >> ~/.bashrc                # Install bash completion
runpod completion zsh >> ~/.zshrc                  # Install zsh completion
```

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
