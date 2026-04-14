---
name: companion-clis
description: Companion CLIs for Runpod workflows — HuggingFace, GitHub, Docker, and AWS.
allowed-tools: Bash(hf:*), Bash(gh:*), Bash(docker:*), Bash(aws:*)
compatibility: Linux, macOS
metadata:
  author: runpod
  version: "1.0"
license: Apache-2.0
---

# Companion CLIs

Four CLIs commonly needed alongside Runpod: HuggingFace (`hf`), GitHub (`gh`), Docker (`docker`), and AWS (`aws`). Each requires credentials before use.

## HuggingFace CLI

The HuggingFace CLI (`hf`) is used to download models into a local Docker container to test and validate the image before deploying to Runpod.

### Install

```bash
# macOS / Linux (standalone installer — recommended)
curl -LsSf https://hf.co/cli/install.sh | bash

# macOS (Homebrew)
brew install hf

# Any platform (pip)
pip install -U huggingface_hub

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://hf.co/cli/install.ps1 | iex"
```

### Credentials

Get a token at https://huggingface.co/settings/tokens. Use **write** access for uploading; **read** access is sufficient for downloading public or gated models.

```bash
# Option 1: interactive login (saves token to ~/.cache/huggingface/token, optionally to git credential store)
hf auth login

# Option 2: non-interactive (pass token directly, useful in scripts and pod start commands)
hf auth login --token $HF_TOKEN --add-to-git-credential

# Option 3: environment variable (takes precedence over saved token; to revert, unset the variable)
export HF_TOKEN=hf_...
```

```bash
hf auth whoami      # confirm auth and org memberships
hf auth logout      # delete all locally stored tokens
```

### Key Commands

```bash
# Download an entire model repo (to HuggingFace cache)
hf download meta-llama/Llama-3.1-8B

# Download to a specific directory (e.g. a network volume mount point)
hf download meta-llama/Llama-3.1-8B --local-dir /workspace/models/llama-3.1-8b

# Download a single file
hf download meta-llama/Llama-3.1-8B config.json --local-dir /workspace/models/llama-3.1-8b

# Download with glob filters (include/exclude)
hf download stabilityai/stable-diffusion-xl-base-1.0 --include "*.safetensors" --exclude "*.fp16.*"

# Download a dataset or Space
hf download HuggingFaceH4/ultrachat_200k --repo-type dataset
hf download HuggingFaceH4/zephyr-chat --repo-type space

# Download a specific revision (commit hash, branch, or tag)
hf download bigcode/the-stack --repo-type dataset --revision v1.1

# Dry-run: see what would be downloaded without downloading
hf download openai-community/gpt2 --dry-run

# Upload a file or folder to the Hub
# Usage: hf upload [repo_id] [local_path] [path_in_repo]
hf upload my-org/my-model ./checkpoint-final .
hf upload my-org/my-model ./results/metrics.json metrics.json
```

### Troubleshooting

```bash
# Increase download timeout on slow connections (default: 10s)
export HF_HUB_DOWNLOAD_TIMEOUT=30
```

---

## GitHub CLI

The GitHub CLI (`gh`) is used to clone private repositories into a local Docker container for building and testing images before deploying to Runpod.

### Install

```bash
# macOS
brew install gh

# Linux (Debian/Ubuntu)
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
     | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update && sudo apt install gh -y

# Linux (Alpine)
apk add github-cli

# Windows
winget install --id GitHub.cli
```

### SSH Keys

SSH keys authenticate git operations and are also used by HuggingFace. Generate one key and register it with both services.

```bash
# Generate an SSH key (if you don't already have one)
ssh-keygen -t ed25519 -C "your_email@example.com"
# Default location: ~/.ssh/id_ed25519 (private), ~/.ssh/id_ed25519.pub (public)

# Add the key to the SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Upload the public key to GitHub
gh ssh-key add ~/.ssh/id_ed25519.pub --title "my-machine"
```

**HuggingFace:** SSH keys must be added manually via browser at https://huggingface.co/settings/keys — paste the contents of `~/.ssh/id_ed25519.pub`.

### Credentials

```bash
# Interactive login — when prompted, select SSH as the git protocol
gh auth login

# Verify auth
gh auth status
```

### Key Commands

```bash
gh repo clone owner/repo                      # clone a repository over SSH
gh repo clone owner/repo -- --depth 1        # shallow clone
gh pr list                                    # list open PRs
gh pr checkout 42                             # check out PR branch locally
gh release list                               # list releases
gh release download v1.0 --dir ./dist        # download release assets
```

---

## Docker

Docker is used to build and validate container images locally before pushing to Docker Hub. Runpod uses Docker Hub as its default image registry — serverless endpoints, pods, and templates all reference images by their Docker Hub tag. Once an image is pushed, Runpod workers pull it automatically when the endpoint or pod is started.

### Install

**macOS:** Download Docker Desktop from https://docs.docker.com/desktop/setup/install/mac-install/
- Choose the **Apple Silicon** installer for M-series Macs, or **Intel Chip** for older Macs
- Open the DMG, drag Docker to Applications, and launch it

**Windows:** Download Docker Desktop from https://docs.docker.com/desktop/setup/install/windows-install/
- Requires WSL 2 (Windows Subsystem for Linux) — Docker Desktop will prompt you to install it if missing
- Run the installer and follow the setup wizard

**Linux:** See https://docs.docker.com/engine/install/ for distro-specific instructions

```bash
# Linux convenience script (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # allow non-root usage (re-login after)
```

### Credentials

Docker Hub authentication uses a personal access token (PAT), not your account password.

1. Go to https://app.docker.com/accounts/gwesterrunpod/settings/personal-access-tokens
2. Click **Generate new token** — give it a descriptive name, set an expiration, and choose **Read & Write** access
3. Copy the token immediately — it is shown only once

```bash
docker login -u <YOUR_DOCKERHUB_USERNAME>
# When prompted for a password, paste your personal access token
```

Credentials are saved to `~/.docker/config.json` after a successful login.

### Tagging

> **Always use explicit semantic version tags. Never rely on `latest`.**

The `latest` tag does not automatically point to the newest image — it only reflects the last build that was pushed without an explicit tag. If you push `v1.0.0` and then push `v1.0.1` with an explicit tag, `latest` still points to `v1.0.0`. Runpod workers pinned to `latest` will silently pull the wrong version.

Use a tag that uniquely identifies the build: `v1.0.0`, `v1.0.1`, etc.

```bash
# Correct: explicit semantic version tag
docker build --platform linux/amd64 -t myorg/myimage:v1.0.0 .
docker push myorg/myimage:v1.0.0

# Wrong: latest tag is ambiguous and unreliable
docker build -t myorg/myimage:latest .
```

### Docker Hub

Docker Hub is the registry Runpod pulls images from. After pushing, images are visible at https://hub.docker.com/repositories/ and referenceable in Runpod as `username/image:tag`.

Images on Docker Hub can be public (anyone can pull) or private (requires credentials). For private images, register your Docker Hub credentials in Runpod once and they become available to any template:

1. Go to https://console.runpod.io/user/settings → **Container Registry Settings**
2. Add your Docker Hub username and personal access token (the same PAT used for `docker login`)
3. When creating or editing a template, select the saved credential from the dropdown

> Runpod currently only supports `docker login` type credentials for container registry authentication.

### Key Commands

```bash
# Build for Runpod (always --platform=linux/amd64 — pods run on x86 Linux)
docker build --platform=linux/amd64 -t myorg/myimage:v1.0.0 .
docker build --platform=linux/amd64 -t myorg/myimage:v1.0.0 -f Dockerfile.prod .  # specify Dockerfile

# Tag an existing image before pushing (does not duplicate image data)
docker tag myorg/myimage:v1.0.0 myorg/myimage:v1.0.1

# Push to Docker Hub (image becomes available to Runpod as myorg/myimage:v1.0.0)
docker push myorg/myimage:v1.0.0

# Run locally for validation
docker run --rm -it myorg/myimage:v1.0.0 bash
docker run --rm --gpus all myorg/myimage:v1.0.0 bash   # with GPU (requires nvidia-container-toolkit)
docker run --rm -p 8080:80 -e API_KEY=secret myorg/myimage:v1.0.0  # port mapping + env vars

# Debug a running container
docker exec -it <container-id> /bin/bash

# Inspect
docker images                          # list local images
docker ps -a                           # list all containers (including stopped)
docker logs <container-id>             # view container output
docker logs -f <container-id>          # follow logs in real time

# Cleanup
docker rmi myorg/myimage:v1.0.0        # remove an image
docker rm <container-id>               # remove a stopped container
```

> **Platform note:** Always build with `--platform=linux/amd64`. Images built on Apple Silicon (arm64) without this flag will fail to start on Runpod.

---

## AWS CLI

The AWS CLI is used to access Runpod storage over the S3 protocol. Any Runpod product that can mount a Network Volume — pods, clusters, and serverless endpoints — can have its storage accessed this way. The bucket name is the network volume ID.

### Install

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
```

### Credentials

Runpod uses its own S3-compatible API, not AWS. You need a Runpod user ID and S3 API key — not an AWS account.

- **Access key** (`AWS_ACCESS_KEY_ID`): your Runpod user ID — found in the console under Settings > S3 API Keys, in the key description (format: `user_...`)
- **Secret key** (`AWS_SECRET_ACCESS_KEY`): an S3 API key — generate one at Settings > S3 API Keys > Create. Shown only once; save it immediately (format: `rps_...`)

```bash
# Option 1: interactive configure (writes ~/.aws/credentials and ~/.aws/config)
# When prompted: enter user ID as access key, S3 API key as secret. Leave region and output format blank.
aws configure

aws configure list    # verify stored credentials

# Option 2: environment variables (override config files)
export AWS_ACCESS_KEY_ID=user_...
export AWS_SECRET_ACCESS_KEY=rps_...

# To stop using env vars and fall back to config file:
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
```

### Endpoints

Every command requires `--region DATACENTER` and `--endpoint-url https://s3api-DATACENTER.runpod.io/`.

Datacenter IDs by region:

| Region | Datacenter IDs |
|--------|---------------|
| EU | CZ-1, RO-1, IS-1, NO-1 |
| US | CA-2, GA-2, IL-1, KS-2, MD-1, MO-1, MO-2, NC-1, NC-2, NE-1, WA-1 |

The datacenter ID for your network volume is shown in the Runpod console.

### Key Commands

Replace `DATACENTER` with the datacenter ID of your network volume (e.g. `CA-2`) and `NETWORK_VOLUME_ID` with the volume ID (used as the S3 bucket name).

```bash
# List files in a volume
aws s3 ls \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/

# List a subdirectory
aws s3 ls \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/my-folder/

# Upload a file
aws s3 cp local-file.txt \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/

# Download a file
aws s3 cp \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/remote-file.txt ./

# Delete a file
aws s3 rm \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/remote-file.txt

# Sync a local directory to a volume
aws s3 sync local-dir/ \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  s3://NETWORK_VOLUME_ID/remote-dir/
```

Path mapping: `/workspace/my-folder/file.txt` on a pod = `s3://NETWORK_VOLUME_ID/my-folder/file.txt` via S3.

### Troubleshooting

```bash
# Retry on timeout (large transfers)
export AWS_RETRY_MODE=standard
export AWS_MAX_ATTEMPTS=10

# Extend read timeout for large files (seconds)
aws s3 cp large-file.zip \
  --region DATACENTER \
  --endpoint-url https://s3api-DATACENTER.runpod.io/ \
  --cli-read-timeout 7200 \
  s3://NETWORK_VOLUME_ID/
```
