---
name: flash
description: Complete knowledge of the runpod-flash framework - SDK, CLI, architecture, deployment, and codebase. Use when working with runpod-flash code, writing @remote functions, configuring resources, debugging deployments, or understanding the framework internals. Triggers on "flash", "runpod-flash", "@remote", "serverless", "deploy", "LiveServerless", "LoadBalancer", "GpuGroup".
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash
---

# Runpod Flash - Complete Framework Knowledge

**runpod-flash** (v1.0.0) is a Python SDK for distributed execution of AI workloads on RunPod's serverless infrastructure. Write Python functions locally, decorate with `@remote`, and Flash handles GPU/CPU provisioning, dependency management, and data transfer.

- **Package**: `pip install runpod-flash`
- **Import**: `from runpod_flash import remote, LiveServerless, GpuGroup, ...`
- **CLI**: `flash`
- **Python**: >=3.10, <3.15

## Getting Started

### 1. Install Flash

```bash
pip install runpod-flash
```

### 2. Set your RunPod API key

Get a key from [RunPod account settings](https://docs.runpod.io/get-started/api-keys), then either export it:

```bash
export RUNPOD_API_KEY=your_api_key_here
```

Or save in a `.env` file in your project directory:

```bash
echo "RUNPOD_API_KEY=your_api_key_here" > .env
```

### 3. Write and run a remote function

```python
import asyncio
from runpod_flash import remote, LiveServerless

gpu_config = LiveServerless(name="my-first-worker")

@remote(resource_config=gpu_config, dependencies=["torch"])
async def gpu_task(data):
    import torch
    tensor = torch.tensor(data, device="cuda")
    return {"sum": tensor.sum().item(), "gpu": torch.cuda.get_device_name(0)}

async def main():
    result = await gpu_task([1, 2, 3, 4, 5])
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
python your_script.py
```

First run takes ~1 minute (endpoint provisioning). Subsequent runs take ~1 second.

### 4. Or create a Flash API project

```bash
flash init my_project        # Generate project template
cd my_project
pip install -r requirements.txt
# Edit .env and add your RUNPOD_API_KEY
flash run                    # Start local FastAPI server at localhost:8888
flash run --auto-provision   # Pre-deploy all endpoints (faster testing)
```

API explorer available at `http://localhost:8888/docs`.

### 5. Build and deploy to production

```bash
flash build                              # Scan @remote functions, package artifact
flash build --exclude torch,torchvision  # Exclude packages in base image (500MB limit)
flash deploy new production              # Create deployment environment
flash deploy send production             # Upload and deploy
flash deploy list                        # List environments
flash deploy info production             # Show details
flash deploy delete production           # Tear down
```

For the full getting started guide with examples, see [references/GETTING_STARTED.md](references/GETTING_STARTED.md).

## Core Concept: The @remote Decorator

The `@remote` decorator marks functions for remote execution on RunPod infrastructure. Code inside runs remotely; code outside runs locally.

```python
from runpod_flash import remote, LiveServerless

config = LiveServerless(name="my-worker")

@remote(resource_config=config, dependencies=["torch", "numpy"])
async def gpu_compute(data):
    import torch  # MUST import inside function
    tensor = torch.tensor(data, device="cuda")
    return {"result": tensor.sum().item()}

result = await gpu_compute([1, 2, 3])
```

### @remote Signature

```python
def remote(
    resource_config: ServerlessResource,  # Required: GPU/CPU config
    dependencies: list[str] = None,       # pip packages
    system_dependencies: list[str] = None,# apt-get packages
    accelerate_downloads: bool = True,    # CDN acceleration
    local: bool = False,                  # Execute locally (testing)
    method: str = None,                   # HTTP method (LoadBalancer only)
    path: str = None,                     # HTTP path (LoadBalancer only)
)
```

### CRITICAL: Cloudpickle Scoping Rules

Functions decorated with `@remote` are serialized with cloudpickle. They can ONLY access:
- Function parameters
- Local variables defined inside the function
- Imports done inside the function
- Built-in Python functions

They CANNOT access: module-level imports, global variables, external functions/classes.

```python
# WRONG - external references
import torch
@remote(resource_config=config)
async def bad(data):
    return torch.tensor(data)  # torch not accessible

# CORRECT - everything inside
@remote(resource_config=config, dependencies=["torch"])
async def good(data):
    import torch
    return torch.tensor(data)
```

### Return Behavior
- Decorated function is always awaitable (`await my_func(...)`)
- Queue-based resources return `JobOutput` with `.output`, `.error`, `.status`
- Load-balanced resources return your dict directly

## Resource Configuration Classes

Choose based on execution model and environment:

| Class | Queue | HTTP | Environment | Use Case |
|-------|-------|------|-------------|----------|
| `LiveServerless` | Yes | No | Dev | GPU with retries, remote code exec |
| `CpuLiveServerless` | Yes | No | Dev | CPU with retries, remote code exec |
| `ServerlessEndpoint` | Yes | No | Prod | GPU, custom Docker images |
| `CpuServerlessEndpoint` | Yes | No | Prod | CPU, custom Docker images |
| `LiveLoadBalancer` | No | Yes | Dev | GPU low-latency HTTP APIs |
| `CpuLiveLoadBalancer` | No | Yes | Dev | CPU low-latency HTTP APIs |
| `LoadBalancerSlsResource` | No | Yes | Prod | GPU production HTTP |
| `CpuLoadBalancerSlsResource` | No | Yes | Prod | CPU production HTTP |

**Queue-based**: Best for batch, long-running tasks, automatic retries.
**Load-balanced**: Best for real-time APIs, low-latency, direct HTTP routing.

**Live* classes**: Fixed optimized Docker image, full remote code execution.
**Non-Live classes**: Custom Docker images, dictionary payload only.

### Common Parameters

```python
LiveServerless(
    name="worker-name",              # Required, unique
    gpus=[GpuGroup.AMPERE_80],       # GPU type(s)
    workersMin=0,                     # Min workers
    workersMax=3,                     # Max workers
    idleTimeout=300,                  # Seconds before scale-down
    networkVolumeId="vol_abc123",     # Persistent storage
    env={"KEY": "value"},             # Environment variables
    template=PodTemplate(containerDiskInGb=100),
)
```

### GPU Groups (GpuGroup enum)

- `GpuGroup.ANY` - Any available (not for production)
- `GpuGroup.AMPERE_16` - RTX A4000, 16GB
- `GpuGroup.AMPERE_24` - RTX A5000, 24GB
- `GpuGroup.AMPERE_48` - A40/RTX A6000, 48GB
- `GpuGroup.AMPERE_80` - A100, 80GB
- `GpuGroup.ADA_24` - RTX 4090, 24GB
- `GpuGroup.ADA_32_PRO` - RTX 5090, 32GB
- `GpuGroup.ADA_48_PRO` - RTX 6000 Ada, 48GB
- `GpuGroup.ADA_80_PRO` - H100, 80GB
- `GpuGroup.HOPPER_141` - H200, 141GB

### CPU Instance Types (CpuInstanceType enum)

Format: `CPU{gen}{type}_{vcpu}_{memory_gb}` (e.g., `CPU5C_4_8`)
- Generations: 3, 5
- Types: G (general), C (compute-optimized)
- Use with `instanceIds` parameter on LiveServerless

## CLI Commands

| Command | Description |
|---------|-------------|
| `flash init [name]` | Initialize project with template |
| `flash run [--auto-provision]` | Start local FastAPI dev server |
| `flash build [--exclude pkg1,pkg2] [--keep-build] [--preview]` | Build deployment artifact |
| `flash deploy new <env>` | Create deployment environment |
| `flash deploy send <env>` | Deploy archive to environment |
| `flash deploy list` | List environments |
| `flash deploy info <env>` | Show environment details |
| `flash deploy delete <env>` | Delete environment |
| `flash undeploy [name\|list]` | Undeploy resources |
| `flash env list\|create\|info\|delete` | Environment management |
| `flash app list\|get` | App management |

### Build Process

1. **Scan**: Find `@remote` decorated functions
2. **Group**: Group functions by `resource_config`
3. **Manifest**: Create `flash_manifest.json` (function-to-endpoint map)
4. **Dependencies**: Install for Linux x86_64
5. **Package**: Bundle into `.flash/artifact.tar.gz`

Build artifacts: `.flash/.build/`, `.flash/artifact.tar.gz`, `.flash/flash_manifest.json`

**500MB deployment limit** - use `--exclude` for packages in base image.

## Architecture Overview

### Deployment Architecture

**Mothership Pattern**: Coordinator endpoint + distributed child endpoints.

1. `flash build` scans code, creates manifest + archive
2. `flash deploy send` uploads archive, provisions resources
3. Mothership boots, reconciles desired vs current state
4. Child endpoints query State Manager GraphQL for service discovery (peer-to-peer)
5. Functions route locally or remotely based on manifest

**Key env vars**:
- `FLASH_IS_MOTHERSHIP=true` - Mothership role
- `FLASH_RESOURCE_NAME` - Child endpoint identity
- `RUNPOD_API_KEY` - API authentication
- `RUNPOD_ENDPOINT_ID` - Set by RunPod platform

### Cross-Endpoint Routing

Functions on different endpoints can call each other transparently:
1. `ProductionWrapper` intercepts calls
2. `ServiceRegistry` looks up function in manifest
3. Local function? Execute directly
4. Remote function? Serialize args (cloudpickle), POST to remote endpoint

**Serialization**: cloudpickle + base64, max 10MB payload.

## Common Gotchas

1. **External scope in @remote functions** - Most common error. Everything must be inside.
2. **Forgetting `await`** - All remote functions must be awaited.
3. **Undeclared dependencies** - Must be in `dependencies=[]` parameter.
4. **Queue vs LB confusion** - Queue returns `JobOutput`, LB returns dict directly.
5. **Large serialization** - Pass URLs/paths, not large data objects.
6. **Imports at module level** - Import inside `@remote` functions, not at top of file.
7. **LoadBalancer requires method+path** - `@remote(config, method="POST", path="/api/x")`

## Detailed References

For more details, see the reference files in this skill:
- [SDK API Reference](references/SDK_API.md) - Complete @remote signature, resource classes, GPU/CPU types, patterns
- [Architecture & Deployment](references/ARCHITECTURE.md) - Deployment flow, manifest system, state management
- [CLI Commands](references/CLI_COMMANDS.md) - All CLI commands with examples

