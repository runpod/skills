---
name: runpodctl
description: Runpod CLI to manage your GPU workloads.
allowed-tools: Bash(runpodctl:*)
compatibility: Linux, macOS
metadata:
  author: runpod
  version: "2.1"
license: Apache-2.0
---

# Runpodctl

Manage GPU pods, serverless endpoints, templates, volumes, and models.

> **Spelling:** "Runpod" (capital R). Command is `runpodctl` (lowercase).

## Install

```bash
# Any platform (official installer)
curl -sSL https://cli.runpod.net | bash

# macOS (Homebrew)
brew install runpod/runpodctl/runpodctl

# macOS (manual — universal binary)
mkdir -p ~/.local/bin && curl -sL https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-darwin-all.tar.gz | tar xz -C ~/.local/bin

# Linux
mkdir -p ~/.local/bin && curl -sL https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-linux-amd64.tar.gz | tar xz -C ~/.local/bin

# Windows (PowerShell)
Invoke-WebRequest -Uri https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-windows-amd64.zip -OutFile runpodctl.zip; Expand-Archive runpodctl.zip -DestinationPath $env:LOCALAPPDATA\runpodctl; [Environment]::SetEnvironmentVariable('Path', $env:Path + ";$env:LOCALAPPDATA\runpodctl", 'User')
```

Ensure `~/.local/bin` is on your `PATH` (add `export PATH="$HOME/.local/bin:$PATH"` to `~/.bashrc` or `~/.zshrc`).

## Quick start

```bash
runpodctl doctor                    # First time setup (API key + SSH)
runpodctl gpu list                  # See available GPUs
runpodctl template search pytorch   # Find a template
runpodctl pod create --template-id runpod-torch-v21 --gpu-id "NVIDIA GeForce RTX 4090"  # Create from template
runpodctl pod list                  # List your pods
```

API key: https://runpod.io/console/user/settings

## Commands

### Pods

```bash
runpodctl pod list                                    # List running pods (default, like docker ps)
runpodctl pod list --all                              # List all pods including exited
runpodctl pod list --status exited                    # Filter by status (RUNNING, EXITED, etc.)
runpodctl pod list --since 24h                        # Pods created within last 24 hours
runpodctl pod list --created-after 2025-01-15         # Pods created after date
runpodctl pod get <pod-id>                            # Get pod details (includes SSH info)
runpodctl pod create --template-id runpod-torch-v21 --gpu-id "NVIDIA GeForce RTX 4090"  # Create from template
runpodctl pod create --image "runpod/pytorch:1.0.3-cu1281-torch291-ubuntu2404" --gpu-id "NVIDIA GeForce RTX 4090"  # Create with image
runpodctl pod create --compute-type cpu --image ubuntu:22.04  # Create CPU pod
runpodctl pod start <pod-id>                          # Start stopped pod
runpodctl pod stop <pod-id>                           # Stop running pod
runpodctl pod restart <pod-id>                        # Restart pod
runpodctl pod reset <pod-id>                          # Reset pod
runpodctl pod update <pod-id> --name "new"            # Update pod
runpodctl pod delete <pod-id>                         # Delete pod (aliases: rm, remove)
```

**List flags:** `--all` / `-a`, `--status`, `--since`, `--created-after`, `--name`, `--compute-type`
**Get flags:** `--include-machine`, `--include-network-volume`

**Create flags:** `--template-id` (required if no `--image`), `--image` (required if no `--template-id`), `--name`, `--gpu-id`, `--gpu-count`, `--compute-type`, `--ssh` (default true), `--container-disk-in-gb`, `--volume-in-gb`, `--volume-mount-path`, `--network-volume-id`, `--ports`, `--env`, `--cloud-type`, `--data-center-ids`, `--global-networking`, `--public-ip`

### Serverless (alias: sls)

```bash
runpodctl serverless list                             # List all endpoints
runpodctl serverless get <endpoint-id>                # Get endpoint details
runpodctl serverless create --name "x" --template-id "tpl_abc"  # Create endpoint
runpodctl serverless update <endpoint-id> --workers-max 5       # Update endpoint
runpodctl serverless delete <endpoint-id>             # Delete endpoint
```

**List flags:** `--include-template`, `--include-workers`
**Update flags:** `--name`, `--workers-min`, `--workers-max`, `--idle-timeout`, `--scaler-type` (QUEUE_DELAY or REQUEST_COUNT), `--scaler-value`
**Create flags:** `--name`, `--template-id`, `--gpu-id`, `--gpu-count`, `--compute-type`, `--workers-min`, `--workers-max`, `--network-volume-id`, `--data-center-ids`

### Templates (alias: tpl)

```bash
runpodctl template list                               # Official + community (first 10)
runpodctl template list --type official               # All official templates
runpodctl template list --type community              # Community templates (first 10)
runpodctl template list --type user                   # Your own templates
runpodctl template list --all                         # Everything including user
runpodctl template list --limit 50                    # Show 50 templates
runpodctl template search pytorch                     # Search for "pytorch" templates
runpodctl template search comfyui --limit 5           # Search, limit to 5 results
runpodctl template search vllm --type official        # Search only official
runpodctl template get <template-id>                  # Get template details (includes README, env, ports)
runpodctl template create --name "x" --image "img"    # Create template
runpodctl template create --name "x" --image "img" --serverless  # Create serverless template
runpodctl template update <template-id> --name "new"  # Update template
runpodctl template delete <template-id>               # Delete template
```

**List flags:** `--type` (official, community, user), `--limit`, `--offset`, `--all`
**Create flags:** `--name`, `--image`, `--container-disk-in-gb`, `--volume-in-gb`, `--volume-mount-path`, `--ports`, `--env`, `--docker-start-cmd`, `--docker-entrypoint`, `--serverless`, `--readme`

### Network Volumes (alias: nv)

```bash
runpodctl network-volume list                         # List all volumes
runpodctl network-volume get <volume-id>              # Get volume details
runpodctl network-volume create --name "x" --size 100 --data-center-id "US-GA-1"  # Create volume
runpodctl network-volume update <volume-id> --name "new"  # Update volume
runpodctl network-volume delete <volume-id>           # Delete volume
```

**Create flags:** `--name`, `--size`, `--data-center-id`

### Models

```bash
runpodctl model list                                  # List your models
runpodctl model list --all                            # List all models
runpodctl model list --name "llama"                   # Filter by name
runpodctl model list --provider "meta"                # Filter by provider
runpodctl model add --name "my-model" --model-path ./model  # Add model
runpodctl model remove --name "my-model"              # Remove model
```

### Registry (alias: reg)

```bash
runpodctl registry list                               # List registry auths
runpodctl registry get <registry-id>                  # Get registry auth
runpodctl registry create --name "x" --username "u" --password "p"  # Create registry auth
runpodctl registry delete <registry-id>               # Delete registry auth
```

### Info

```bash
runpodctl user                                        # Account info and balance (alias: me)
runpodctl gpu list                                    # List available GPUs
runpodctl gpu list --include-unavailable              # Include unavailable GPUs
runpodctl datacenter list                             # List datacenters (alias: dc)
runpodctl billing pods                                # Pod billing history
runpodctl billing serverless                          # Serverless billing history
runpodctl billing network-volume                      # Volume billing history
```

### SSH

```bash
runpodctl ssh info <pod-id>                           # Get SSH info (command + key, does not connect)
runpodctl ssh list-keys                               # List SSH keys
runpodctl ssh add-key                                 # Add SSH key
```

**Agent note:** `ssh info` returns connection details, not an interactive session. If interactive SSH is not available, execute commands remotely via `ssh user@host "command"`.

### File Transfer

**Agent instructions (send/receive files or directories):**

1. **Sender (local or remote):** Start the sending side **without** `--code`:

   ```bash
   runpodctl send [path to file or directory]
   ```

   **Session key:** Read stdout and treat the **first line** as the secret code for this transfer only. Ignore subsequent lines unless you need logs. Do not invent or reuse a code; each `send` run produces a fresh key on that first line. After printing the code, `send` typically blocks until the receiver connects—capture the first line promptly (for example run `send` where stdout can be read as it arrives, or tee to a log), then start `receive` on the other side with that exact string.

2. **Receiver (local or remote):** Pass that same secret as the first positional argument (there is no `--code` flag on `receive`):

   ```bash
   runpodctl receive [secret from send's first line of stdout]
   ```

   On whichever machine is not running the command directly, use `runpodctl ssh` (into the pod or host) to start the matching end there—one side runs `send`, the other runs `receive`, both using the identical secret from step 1.

3. **Completion:** Wait until both processes exit with status code `0`. If either exits non-zero, diagnose and retry by running **`send` again** and using the **new** first line of output as the secret (do not reuse the previous transfer's code).

4. **Do not pre-process paths:** `runpodctl` handles incremental transfer (changed files), encryption, and compression. Do not compress, encrypt, or otherwise alter the source tree before sending.

Example script you can reference. You can make changes as needed. 
```
  #!/usr/bin/env bash
  # Upload a file or directory to a Runpod pod.
  # Usage: upload.sh <pod-id> <local-path>

  set -euo pipefail

  POD_ID="${1:?Usage: $0 <pod-id> <local-path>}"
  SEND_PATH="${2:?Usage: $0 <pod-id> <local-path>}"

  # 1. Get SSH connection details
  SSH_INFO=$(runpodctl ssh info "$POD_ID")
  SSH_HOST=$(echo "$SSH_INFO" | awk -F'"' '/"ip"/{print $4}')
  SSH_PORT=$(echo "$SSH_INFO" | awk '/"port"/{gsub(/[^0-9]/,"",$0); print}')
  SSH_KEY=$(echo "$SSH_INFO"  | awk -F'"' '/"path"/{print $4; exit}')

  # 2. Start sender in background; capture output to extract nonce from first line
  TMPLOG=$(mktemp)
  runpodctl send "$SEND_PATH" >"$TMPLOG" 2>&1 &
  SEND_PID=$!

  # 3. Wait for the nonce (first line of sender output)
  NONCE=""
  for i in $(seq 1 15); do
    sleep 1
    NONCE=$(head -1 "$TMPLOG" 2>/dev/null)
    [ -n "$NONCE" ] && break
  done

  if [ -z "$NONCE" ]; then
    echo "ERROR: timed out waiting for nonce" >&2
    kill $SEND_PID 2>/dev/null; rm -f "$TMPLOG"; exit 1
  fi

  echo "Nonce: $NONCE"

  # 4. Run receiver on the pod
  ssh -i "$SSH_KEY" "root@$SSH_HOST" -p "$SSH_PORT" -o StrictHostKeyChecking=no \
    "runpodctl receive $NONCE"
  RECV_EXIT=$?

  # 5. Wait for sender to finish
  wait $SEND_PID; SEND_EXIT=$?
  rm -f "$TMPLOG"

  if [ "$SEND_EXIT" -ne 0 ] || [ "$RECV_EXIT" -ne 0 ]; then
    echo "ERROR: transfer failed (send=$SEND_EXIT, recv=$RECV_EXIT)" >&2
    exit 1
  fi

  echo "Transfer complete."
```

### Utilities

```bash
runpodctl doctor                                      # Diagnose and fix CLI issues
runpodctl update                                      # Update CLI
runpodctl version                                     # Show version
runpodctl completion                                  # Auto-detect shell and install completion
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
