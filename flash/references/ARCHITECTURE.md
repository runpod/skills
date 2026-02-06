# Flash Architecture & Deployment Reference

## Deployment Architecture

### Mothership + Child Endpoints Pattern

Flash Deploy uses a coordinator (Mothership) + distributed worker (Child Endpoint) pattern:

- **Mothership**: Orchestration endpoint responsible for reconciliation and state management
- **Child Endpoints**: Worker endpoints executing `@remote` functions, one per resource config
- **State Manager**: RunPod GraphQL API for persistent state and peer-to-peer service discovery

### Build-Deploy Flow

1. **`flash build`** (local):
   - `RemoteDecoratorScanner` scans Python files for `@remote` decorators
   - `ManifestBuilder` creates `flash_manifest.json`
   - Dependencies installed for Linux x86_64
   - Everything packaged into `.flash/artifact.tar.gz`

2. **`flash deploy new <env>`** (local):
   - Creates FlashApp in RunPod (if first environment)
   - Creates FlashEnvironment with the specified name
   - Provisions mothership serverless endpoint

3. **`flash deploy send <env>`** (local):
   - Uploads `artifact.tar.gz` to S3
   - Provisions all child endpoints upfront
   - Environment activated with archive reference

4. **Mothership Boot** (cloud):
   - Loads manifest from `.flash/` directory
   - `MothershipsProvisioner.reconcile_children()` runs
   - Computes diff: new, changed, removed, unchanged resources
   - Updates State Manager with reconciliation results

5. **Child Endpoint Boot** (cloud):
   - Reads manifest from `.flash/` directory
   - `ServiceRegistry` builds function-to-endpoint mapping
   - Queries State Manager for peer endpoint URLs (peer-to-peer)
   - Ready to execute functions

### Manifest System

**Build-time manifest** (`flash_manifest.json`):
```json
{
  "version": "1.0",
  "generated_at": "2026-01-12T10:30:00Z",
  "project_name": "my-app",
  "function_registry": {
    "funcA": "gpu_config",
    "funcB": "cpu_config"
  },
  "resources": {
    "gpu_config": {
      "resource_type": "LiveServerless",
      "functions": [
        {"name": "funcA", "module": "app.handlers", "is_async": true, "is_class": false}
      ],
      "is_load_balanced": false
    }
  },
  "routes": {}
}
```

**Runtime persisted manifest** (State Manager) adds:
- `config_hash` per resource
- `endpoint_url` per resource
- `status` per resource (deployed, updated, failed)

### Reconciliation Logic

`MothershipsProvisioner`:

1. Load local manifest from `.flash/` (desired state)
2. Fetch persisted state from State Manager (previous state)
3. Compute `ManifestDiff`:
   - `new`: In local but not persisted -> Deploy
   - `changed`: Different `config_hash` -> Update
   - `removed`: In persisted but not local -> Undeploy
   - `unchanged`: Same `config_hash` -> Skip
4. Execute changes (parallel, max 3 concurrent)
5. Persist new state to State Manager

### State Management

**Local State**: `.runpod/resources.pkl`
- Format: `(Dict[str, DeployableResource], Dict[str, str])` (resources, config_hashes)
- Protected by cross-platform file locking
- Used by `ResourceManager` singleton

**Remote State**: RunPod State Manager (GraphQL API)
- Persists deployment state across boots
- Peer-to-peer service discovery
- Thread-safe with `asyncio.Lock`
- 3 GQL roundtrips per update

### Cross-Endpoint Routing

1. `ProductionWrapper` intercepts function call at stub layer
2. `ServiceRegistry` loads manifest and looks up function
3. If function's resource == current endpoint: execute locally
4. If function's resource != current endpoint: serialize and POST remotely

**Serialization**: cloudpickle + base64, max 10MB
**Caching**: Manifest cached 300s, endpoint URLs cached 300s

### Environment Variables

| Variable | Where | Required | Purpose |
|----------|-------|----------|---------|
| `FLASH_IS_MOTHERSHIP` | Mothership | Yes | Identifies mothership role |
| `FLASH_RESOURCE_NAME` | Child | Yes | Resource config identity |
| `RUNPOD_API_KEY` | All | Yes | API authentication |
| `RUNPOD_ENDPOINT_ID` | All | Auto-set | Endpoint identification |
| `FLASH_MANIFEST_PATH` | All | No | Override manifest location |

### Flash Apps & Environments

- **FlashApp**: Top-level container per project
- **FlashEnvironment**: Named deployment target (e.g., "production", "staging")
- **FlashBuild**: Archive artifact uploaded per app

Lifecycle:
1. `FlashApp` hydrates (fetches/creates remote app ID)
2. `create_environment_and_app()` ensures app exists + creates environment
3. `flash deploy send` uploads build and provisions resources
4. `flash deploy delete` undeploys resources then deletes environment

### Reliability Features

- **Circuit Breaker**: Per-endpoint circuit breakers with sliding window
- **Load Balancer**: Multiple strategies for endpoint selection
- **Retry Manager**: Exponential backoff with configurable max attempts
- **Metrics**: Structured logging for circuit breaker, retry, and LB events

Configuration via `ReliabilityConfig`:
```python
ReliabilityConfig(
    circuit_breaker=CircuitBreakerConfig(failure_threshold=5, recovery_timeout=30),
    load_balancer=LoadBalancerConfig(strategy=LoadBalancerStrategy.ROUND_ROBIN),
    retry=RetryConfig(max_retries=3, backoff_factor=2.0),
    metrics=MetricsConfig(enabled=True),
)
```
