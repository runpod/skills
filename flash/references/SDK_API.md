# Flash SDK API Reference

## Public Exports

All imports from `runpod_flash`:

```python
from runpod_flash import (
    remote,                        # @remote decorator
    LiveServerless,                # Dev GPU queue-based (full remote exec)
    CpuLiveServerless,             # Dev CPU queue-based (full remote exec)
    ServerlessEndpoint,            # Prod GPU queue-based (custom Docker)
    CpuServerlessEndpoint,         # Prod CPU queue-based (custom Docker)
    LiveLoadBalancer,              # Dev GPU load-balanced (full remote exec)
    CpuLiveLoadBalancer,           # Dev CPU load-balanced (full remote exec)
    LoadBalancerSlsResource,       # Prod GPU load-balanced (custom Docker)
    CpuLoadBalancerSlsResource,    # Prod CPU load-balanced (custom Docker)
    GpuGroup,                      # GPU type enum
    GpuType,                       # Individual GPU type enum
    CpuInstanceType,               # CPU instance type enum
    NetworkVolume,                 # Persistent storage
    PodTemplate,                   # Pod configuration
    CudaVersion,                   # CUDA version enum
    DataCenter,                    # Data center enum
    ServerlessType,                # QB vs LB enum
    ResourceManager,               # Resource provisioning singleton
    FlashApp,                      # Flash app management
)
```

## @remote Decorator - Complete Reference

**Location**: `src/runpod_flash/client.py`

```python
def remote(
    resource_config: ServerlessResource,
    dependencies: Optional[List[str]] = None,
    system_dependencies: Optional[List[str]] = None,
    accelerate_downloads: bool = True,
    local: bool = False,
    method: Optional[str] = None,
    path: Optional[str] = None,
    **extra,
)
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `resource_config` | `ServerlessResource` | Required | GPU/CPU and scaling config |
| `dependencies` | `list[str]` | `None` | pip packages to install |
| `system_dependencies` | `list[str]` | `None` | apt-get packages to install |
| `accelerate_downloads` | `bool` | `True` | CDN acceleration for downloads |
| `local` | `bool` | `False` | Execute locally (skip remote) |
| `method` | `str` | `None` | HTTP method (LoadBalancer only): GET, POST, PUT, DELETE, PATCH |
| `path` | `str` | `None` | HTTP path (LoadBalancer only): must start with "/" |

### Behavior

1. If running on RunPod platform (`RUNPOD_POD_ID` or `RUNPOD_ENDPOINT_ID` set): returns function unchanged with `__remote_config__` attached
2. If `local=True`: returns function unchanged
3. If class: wraps with `create_remote_class()`
4. If function: wraps in async wrapper that:
   - Gets or deploys resource via `ResourceManager`
   - Creates stub from resource
   - Executes function via stub

### LoadBalancer Validation

When `resource_config` is `LoadBalancerSlsResource`:
- `method` and `path` are **required**
- `path` must start with "/"
- `method` must be one of: GET, POST, PUT, DELETE, PATCH

### Function Requirements

1. **Cloudpickle scoping**: All imports, helpers, and variables must be defined inside the function
2. **Always awaitable**: Even sync functions become async after decoration
3. **Serializable args**: Arguments must be cloudpickle-compatible
4. **JSON-serializable returns**: Return dicts, lists, strings, numbers
5. **Dependencies declared**: All pip packages must be in `dependencies` list

## Resource Configuration - Detailed

### ServerlessResource (Base)

**Location**: `src/runpod_flash/core/resources/serverless.py`

```python
class ServerlessResource(DeployableResource):
    name: str                     # Required, unique endpoint name
    gpus: list = [GpuGroup.ANY]   # GPU types
    gpuCount: int = 1             # GPUs per worker
    workersMin: int = 0           # Min workers
    workersMax: int = 3           # Max workers
    idleTimeout: int = 5          # Minutes before scale-down
    type: ServerlessType = QB     # QB or LB
    scalerType: ServerlessScalerType = QUEUE_DELAY
    scalerValue: int = 4          # Scaler parameter
    locations: str = None         # Datacenter preference
    networkVolumeId: str = None   # Network volume ID
    env: dict = None              # Environment variables
    template: PodTemplate = None  # Pod template override
    executionTimeoutMs: int = 0   # Max execution time (0 = no limit)
    imageName: str = ...          # Docker image
```

### LiveServerless

**Location**: `src/runpod_flash/core/resources/live_serverless.py`

Extends `ServerlessEndpoint` with `LiveServerlessMixin`:
- Fixed optimized Docker image (cannot be overridden)
- Full remote Python function execution via cloudpickle
- Development-focused with spot pricing

```python
config = LiveServerless(
    name="my-gpu-worker",
    gpus=[GpuGroup.AMPERE_80],
    workersMin=0,
    workersMax=5,
    idleTimeout=10,
    template=PodTemplate(containerDiskInGb=100),
    env={"HF_TOKEN": "xyz"},
)
```

### LoadBalancerSlsResource

**Location**: `src/runpod_flash/core/resources/load_balancer_sls_resource.py`

Key differences from ServerlessResource:
- Always `type=LB` (enforced, cannot override)
- Always `scalerType=REQUEST_COUNT` (required)
- Health checks via `/ping` endpoint
- Supports custom HTTP routes

```python
@remote(
    resource_config=LiveLoadBalancer(name="api"),
    method="POST",
    path="/api/process",
)
async def handler(x: int, y: int):
    return {"result": x + y}
```

### CPU Resources

Use `instanceIds` parameter with `CpuInstanceType`:

```python
config = LiveServerless(
    name="cpu-worker",
    instanceIds=[CpuInstanceType.CPU5C_4_8],  # Forces CPU endpoint
    workersMax=5,
)
```

Or use explicit CPU classes:
```python
from runpod_flash import CpuLiveServerless
config = CpuLiveServerless(name="cpu-worker", workersMax=5)
```

### PodTemplate

**Location**: `src/runpod_flash/core/resources/template.py`

Override pod-level settings:

```python
from runpod_flash import PodTemplate

template = PodTemplate(
    containerDiskInGb=100,
    env=[{"key": "PYTHONPATH", "value": "/workspace"}],
)

config = LiveServerless(name="worker", template=template)
```

### NetworkVolume

**Location**: `src/runpod_flash/core/resources/network_volume.py`

```python
from runpod_flash import NetworkVolume, DataCenter

volume = NetworkVolume(
    name="model-storage",
    size=100,  # GB
    dataCenterId=DataCenter.EU_RO_1,
)
```

## GPU Groups - Complete Reference

**Location**: `src/runpod_flash/core/resources/gpu.py`

```python
class GpuGroup(Enum):
    ANY            # Any available GPU
    AMPERE_16      # RTX A4000, 16GB VRAM
    AMPERE_24      # RTX A5000/L4/RTX 3090, 24GB VRAM
    AMPERE_48      # A40/RTX A6000, 48GB VRAM
    AMPERE_80      # A100, 80GB VRAM
    ADA_24         # RTX 4090, 24GB VRAM
    ADA_32_PRO     # RTX 5090, 32GB VRAM
    ADA_48_PRO     # RTX 6000 Ada, 48GB VRAM
    ADA_80_PRO     # H100, 80GB VRAM
    HOPPER_141     # H200, 141GB VRAM
```

## CPU Instance Types - Complete Reference

**Location**: `src/runpod_flash/core/resources/cpu.py`

Format: `CPU{generation}{type}_{vcpu}_{memory_gb}`

| Instance Type | Gen | Type | vCPU | RAM |
|--------------|-----|------|------|-----|
| `CPU3G_1_4` | 3rd | General | 1 | 4GB |
| `CPU3G_2_8` | 3rd | General | 2 | 8GB |
| `CPU3G_4_16` | 3rd | General | 4 | 16GB |
| `CPU3G_8_32` | 3rd | General | 8 | 32GB |
| `CPU3C_1_2` | 3rd | Compute | 1 | 2GB |
| `CPU3C_2_4` | 3rd | Compute | 2 | 4GB |
| `CPU3C_4_8` | 3rd | Compute | 4 | 8GB |
| `CPU3C_8_16` | 3rd | Compute | 8 | 16GB |
| `CPU5C_1_2` | 5th | Compute | 1 | 2GB |
| `CPU5C_2_4` | 5th | Compute | 2 | 4GB |
| `CPU5C_4_8` | 5th | Compute | 4 | 8GB |
| `CPU5C_8_16` | 5th | Compute | 8 | 16GB |

## Error Handling

### Queue-Based Resources

```python
job_output = await my_function(data)
if job_output.error:
    print(f"Failed: {job_output.error}")
else:
    result = job_output.output
```

`JobOutput` fields: `id`, `status`, `output`, `error`, `started_at`, `ended_at`

### Load-Balanced Resources

```python
try:
    result = await my_function(data)  # Returns dict directly
except Exception as e:
    print(f"Error: {e}")
```

### Runtime Exceptions

**Location**: `src/runpod_flash/runtime/exceptions.py`

```
FlashRuntimeError (base)
  RemoteExecutionError      # Remote function failed
  SerializationError        # cloudpickle serialization failed
  GraphQLError              # GraphQL base error
    GraphQLMutationError    # Mutation failed
    GraphQLQueryError       # Query failed
  ManifestError             # Invalid/missing manifest
  ManifestServiceUnavailableError  # State Manager unreachable
```

## Common Patterns

### Hybrid GPU/CPU Pipeline

```python
cpu_config = LiveServerless(name="preprocessor", instanceIds=[CpuInstanceType.CPU5C_4_8])
gpu_config = LiveServerless(name="inference", gpus=[GpuGroup.AMPERE_80])

@remote(resource_config=cpu_config, dependencies=["pandas"])
async def preprocess(data):
    import pandas as pd
    return pd.DataFrame(data).to_dict('records')

@remote(resource_config=gpu_config, dependencies=["torch"])
async def inference(data):
    import torch
    tensor = torch.tensor(data, device="cuda")
    return {"result": tensor.sum().item()}

async def pipeline(raw_data):
    clean = await preprocess(raw_data)
    return await inference(clean)
```

### Parallel Execution

```python
results = await asyncio.gather(
    process_item(item1),
    process_item(item2),
    process_item(item3),
)
```

### Load-Balanced HTTP API

```python
api = LiveLoadBalancer(name="api-service")

@remote(api, method="POST", path="/api/process")
async def process(x: int, y: int):
    return {"result": x + y}

@remote(api, method="GET", path="/api/health")
def health():
    return {"status": "ok"}
```

### Local Testing

```python
@remote(resource_config=config, local=True)
async def my_function(data):
    return {"status": "ok"}  # Runs locally
```

### Cost Optimization

- Use `workersMin=0` to scale from zero
- Use `idleTimeout=600` to reduce churn
- Use smaller GPUs if they fit your model
- Use Live* classes for spot pricing in dev
- Pass URLs/paths instead of large data objects
