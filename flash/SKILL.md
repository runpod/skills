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
- **CLI**: `flash` (entry point: `src/runpod_flash/cli/main.py`)
- **Python**: >=3.10, <3.15
- **Source layout**: `src/runpod_flash/`

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

### Source Structure

```
src/runpod_flash/
  __init__.py              # Public API exports (lazy-loaded)
  client.py                # @remote decorator implementation
  config.py                # FlashPaths configuration
  execute_class.py         # Remote class execution
  logger.py                # Logging setup
  cli/
    main.py                # Typer CLI entry point
    commands/
      init.py, run.py, build.py, deploy.py, undeploy.py, env.py, apps.py
      build_utils/
        scanner.py         # RemoteDecoratorScanner - finds @remote
        manifest.py        # ManifestBuilder - creates flash_manifest.json
        handler_generator.py, lb_handler_generator.py
  core/
    api/runpod.py          # RunpodGraphQLClient, RunpodRestClient
    deployment.py          # DeploymentOrchestrator
    discovery.py           # ResourceDiscovery
    resources/
      base.py              # BaseResource, DeployableResource (ABC)
      serverless.py        # ServerlessResource, ServerlessEndpoint
      serverless_cpu.py    # CpuEndpointMixin, CpuServerlessEndpoint
      live_serverless.py   # LiveServerless, CpuLiveServerless, LiveLoadBalancer, etc.
      load_balancer_sls_resource.py  # LoadBalancerSlsResource
      network_volume.py    # NetworkVolume, DataCenter
      resource_manager.py  # ResourceManager (singleton)
      template.py          # PodTemplate
      gpu.py               # GpuGroup, GpuType enums
      cpu.py               # CpuInstanceType enum
      app.py               # FlashApp
    utils/
      singleton.py, file_lock.py, lru_cache.py, backoff.py
  runtime/
    generic_handler.py     # Queue-based handler factory
    lb_handler.py          # Load-balanced (FastAPI) handler factory
    service_registry.py    # ServiceRegistry - function routing
    manifest_fetcher.py    # ManifestFetcher - loads manifest
    mothership_provisioner.py  # Reconciliation logic
    state_manager_client.py    # GraphQL state persistence
    production_wrapper.py  # Routes local vs remote execution
    serialization.py       # cloudpickle + base64
    circuit_breaker.py, load_balancer.py, retry_manager.py, metrics.py
    reliability_config.py, models.py, config.py, exceptions.py
  stubs/
    live_serverless.py     # LiveServerlessStub
    serverless.py, load_balancer_sls.py, registry.py
  protos/
    remote_execution.py    # FunctionRequest/FunctionResponse models
```

### Class Hierarchy

```
BaseResource (Pydantic BaseModel)
  DeployableResource (ABC)
    ServerlessResource
      ServerlessEndpoint
      LoadBalancerSlsResource
        CpuLoadBalancerSlsResource
    NetworkVolume

LiveServerlessMixin
  LiveServerless (+ ServerlessEndpoint)
  CpuLiveServerless (+ CpuServerlessEndpoint)
  LiveLoadBalancer (+ LoadBalancerSlsResource)
  CpuLiveLoadBalancer (+ CpuLoadBalancerSlsResource)

CpuEndpointMixin
  CpuServerlessEndpoint (+ ServerlessEndpoint)
  CpuLoadBalancerSlsResource (+ LoadBalancerSlsResource)
```

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

## Development Workflow

```bash
make dev          # Install in editable mode
make test-unit    # Run unit tests (parallel)
make test         # Run all tests
make lint         # Ruff check
make format       # Ruff format
make index        # Rebuild code intel index
make build        # Build PyPI package
```

- Tests: `pytest` with `pytest-asyncio`, `pytest-xdist` (parallel), `pytest-cov`
- Coverage threshold: 65%
- Linting: `ruff` (target py310, line-length 88)
- Type checking: `mypy` (lenient config)

## Key Design Patterns

- **Singleton**: `ResourceManager` (thread-safe via `SingletonMixin`)
- **File Locking**: Cross-platform locks for `.runpod/resources.pkl`
- **Config Hashing**: Drift detection via hash comparison
- **Lazy Loading**: `__init__.py` uses `__getattr__` for fast CLI startup
- **Manifest-Driven**: JSON manifest is the contract between build and runtime
- **Peer-to-Peer**: All endpoints query State Manager GraphQL directly (no hub)

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

Or in the codebase docs/ directory:
- `docs/Flash_SDK_Reference.md` - Full SDK reference
- `docs/Flash_Deploy_Guide.md` - Deployment architecture spec
- `docs/Deployment_Architecture.md` - Deployment diagrams
- `docs/Cross_Endpoint_Routing.md` - Cross-endpoint routing
- `docs/Load_Balancer_Endpoints.md` - Load-balanced endpoints
- `docs/Flash_Apps_and_Environments.md` - Apps & environments
- `docs/Using_Remote_With_LoadBalancer.md` - @remote with LB
- `docs/LoadBalancer_Runtime_Architecture.md` - LB runtime details
