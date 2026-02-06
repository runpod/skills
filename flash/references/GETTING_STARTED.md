# Flash Getting Started - Complete Guide

## Prerequisites

- Python 3.10 or higher
- A RunPod account with API key ([sign up](https://runpod.io/console))
- Basic knowledge of Python and async programming

## Installation

```bash
pip install runpod-flash
```

This installs:
- The `runpod_flash` Python package
- The `flash` CLI tool

### Verify installation

```bash
flash --version
# Output: Runpod Flash CLI v1.0.0
```

## API Key Setup

### Option A: Environment variable

```bash
export RUNPOD_API_KEY=your_api_key_here
```

### Option B: .env file (recommended for projects)

Create a `.env` file in your project root:

```
RUNPOD_API_KEY=your_api_key_here
```

Flash automatically loads `.env` files via `python-dotenv`.

### Option C: Inside Flash project template

When using `flash init`, the template includes a `.env` file:

```
RUNPOD_API_KEY=your_api_key_here
# FLASH_HOST=localhost
# FLASH_PORT=8888
# LOG_LEVEL=INFO
```

## Usage Mode 1: Standalone Scripts

Write Python scripts that execute functions on remote GPUs/CPUs.

### Basic GPU Example

```python
import asyncio
from runpod_flash import remote, LiveServerless

gpu_config = LiveServerless(name="flash-quickstart")

@remote(
    resource_config=gpu_config,
    dependencies=["torch", "numpy"]
)
async def gpu_compute(data):
    import torch
    import numpy as np

    tensor = torch.tensor(data, device="cuda")
    result = tensor.sum().item()

    return {
        "result": result,
        "device": torch.cuda.get_device_name(0)
    }

async def main():
    result = await gpu_compute([1, 2, 3, 4, 5])
    print(f"Sum: {result['result']}")
    print(f"Computed on: {result['device']}")

if __name__ == "__main__":
    asyncio.run(main())
```

Run: `python your_script.py`

First run: ~1 minute (endpoint provisioning + cold start).
Subsequent runs: ~1 second.

### Basic CPU Example

```python
import asyncio
from runpod_flash import remote, LiveServerless, CpuInstanceType

cpu_config = LiveServerless(
    name="cpu-worker",
    instanceIds=[CpuInstanceType.CPU5C_2_4],
)

@remote(resource_config=cpu_config, dependencies=["pandas"])
async def process_data(data):
    import pandas as pd
    df = pd.DataFrame(data)
    return {"rows": len(df), "stats": df.describe().to_dict()}

async def main():
    result = await process_data([
        {"name": "Alice", "score": 85},
        {"name": "Bob", "score": 92},
    ])
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

### Parallel Execution

```python
import asyncio

async def main():
    results = await asyncio.gather(
        gpu_compute([1, 2, 3]),
        gpu_compute([4, 5, 6]),
        gpu_compute([7, 8, 9]),
    )
    print(results)
```

### Hybrid GPU + CPU Pipeline

```python
from runpod_flash import remote, LiveServerless, CpuInstanceType

cpu_config = LiveServerless(name="preprocessor", instanceIds=[CpuInstanceType.CPU5C_4_8])
gpu_config = LiveServerless(name="inference")

@remote(resource_config=cpu_config, dependencies=["pandas"])
async def preprocess(raw_data):
    import pandas as pd
    return pd.DataFrame(raw_data).to_dict('records')

@remote(resource_config=gpu_config, dependencies=["torch"])
async def inference(clean_data):
    import torch
    tensor = torch.tensor([[v for v in row.values()] for row in clean_data], device="cuda")
    return {"predictions": tensor.mean(dim=1).tolist()}

async def main():
    clean = await preprocess(raw_data)
    results = await inference(clean)
```

## Usage Mode 2: Flash API Endpoints (FastAPI)

Build and serve HTTP API endpoints backed by remote GPU/CPU workers.

### Step 1: Initialize project

```bash
flash init my_project
cd my_project
```

Creates:
```
my_project/
├── main.py                    # FastAPI application entry point
├── workers/
│   ├── gpu/
│   │   ├── __init__.py        # FastAPI router
│   │   └── endpoint.py        # GPU @remote function
│   └── cpu/
│       ├── __init__.py        # FastAPI router
│       └── endpoint.py        # CPU @remote function
├── .env                       # API key template
├── .gitignore
├── .flashignore               # Deployment ignore patterns
├── requirements.txt
└── README.md
```

### Step 2: Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3: Configure API key

Edit `.env`:
```
RUNPOD_API_KEY=your_api_key_here
```

### Step 4: Start dev server

```bash
flash run
```

Server starts at `http://localhost:8888`.
API explorer at `http://localhost:8888/docs`.

Test with curl:
```bash
curl -X POST http://localhost:8888/gpu/hello \
    -H "Content-Type: application/json" \
    -d '{"message": "Hello from the GPU!"}'
```

### Step 5: Auto-provision for faster testing

```bash
flash run --auto-provision
```

Deploys all endpoints upfront, eliminating cold-start delays.

### Step 6: Customize

1. Add/edit remote functions in `endpoint.py` files
2. Test scripts individually: `python workers/gpu/endpoint.py`
3. Configure routers in `__init__.py` files
4. Add new endpoints to `main.py`

## Build & Deploy to Production

### Step 1: Build

```bash
flash build
```

This:
1. Scans for `@remote` functions
2. Creates `flash_manifest.json`
3. Installs dependencies for Linux x86_64
4. Packages into `.flash/artifact.tar.gz`

**Managing bundle size** (500MB limit):
```bash
# Exclude packages already in base Docker image
flash build --exclude torch,torchvision,torchaudio   # GPU deployments
```

**Local preview with Docker**:
```bash
flash build --preview    # Runs in local Docker containers
```

### Step 2: Create deployment environment

```bash
flash deploy new production
```

### Step 3: Deploy

```bash
flash deploy send production
```

### Step 4: Manage

```bash
flash deploy list                  # List all environments
flash deploy info production       # Show details
flash deploy delete production     # Tear down (double confirmation required)
```

## Load-Balanced HTTP Endpoints

For low-latency REST APIs (no queue):

```python
from runpod_flash import remote, LiveLoadBalancer

api = LiveLoadBalancer(name="api-service")

@remote(api, method="POST", path="/api/process")
async def process_data(x: int, y: int):
    return {"result": x + y}

@remote(api, method="GET", path="/api/health")
def health_check():
    return {"status": "ok"}

result = await process_data(5, 3)  # {"result": 8}
```

Key differences from queue-based:
- Direct HTTP routing (no queue)
- Lower latency
- Returns dict directly (no JobOutput wrapper)
- No automatic retries
- Requires `method` and `path` parameters

## All CLI Commands

| Command | Description |
|---------|-------------|
| `flash --version` | Show version |
| `flash init [name]` | Initialize project template |
| `flash run [--auto-provision]` | Start local dev server |
| `flash build [--exclude X] [--preview]` | Build deployment artifact |
| `flash deploy new <env>` | Create deployment environment |
| `flash deploy send <env>` | Deploy to environment |
| `flash deploy list` | List environments |
| `flash deploy info <env>` | Show environment details |
| `flash deploy delete <env>` | Delete environment |
| `flash undeploy list` | List deployed resources |
| `flash undeploy <name>` | Undeploy specific resource |
| `flash env list\|create\|info\|delete` | Environment management |
| `flash app list\|get` | App management |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `RUNPOD_API_KEY` | Yes | RunPod API key for authentication |
| `FLASH_HOST` | No | Dev server host (default: localhost) |
| `FLASH_PORT` | No | Dev server port (default: 8888) |
| `LOG_LEVEL` | No | Logging level (default: INFO) |

## Common Issues

### First run is slow (~1 min)
Normal. Endpoint is being provisioned. Subsequent runs reuse the endpoint.

### Import errors in remote functions
Imports must be inside the `@remote` function, not at module level:
```python
@remote(resource_config=config, dependencies=["torch"])
async def my_func(data):
    import torch  # Inside, not at top of file
```

### Authentication errors
```bash
echo $RUNPOD_API_KEY  # Verify it's set
```

### Forgetting await
All `@remote` functions must be awaited:
```python
result = await my_function(data)  # Not: result = my_function(data)
```

### Dependencies not declared
Packages used inside `@remote` functions must be listed in `dependencies`:
```python
@remote(resource_config=config, dependencies=["numpy", "pandas"])
async def my_func(data):
    import numpy as np
    import pandas as pd
```

### Bundle too large (>500MB)
Exclude packages pre-installed in the base Docker image:
```bash
flash build --exclude torch,torchvision,torchaudio
```

### Endpoints accumulate in RunPod account
Endpoints persist until deleted. Clean up via:
```bash
flash undeploy list       # See what's deployed
flash undeploy <name>     # Remove specific endpoint
```
Or delete manually in the [RunPod console](https://runpod.io/console).
