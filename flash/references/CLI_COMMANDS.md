# Flash CLI Commands Reference

CLI entry point: `flash` (registered via `pyproject.toml` → `src/runpod_flash/cli/main.py`)

Built with **Typer** + **Rich** for terminal UI.

## flash init

Initialize a new Flash project with template.

```bash
flash init [project_name]
```

- With name: Creates `project_name/` directory with template
- Without name: Initializes current directory

**Template structure**:
```
project_name/
├── main.py                # FastAPI entry point
├── workers/
│   ├── gpu/__init__.py    # GPU router
│   │   └── endpoint.py    # GPU @remote function
│   └── cpu/__init__.py    # CPU router
│       └── endpoint.py    # CPU @remote function
├── .env                   # API key template
├── .gitignore
├── .flashignore           # Deployment ignore patterns
├── requirements.txt
└── README.md
```

**Implementation**: `src/runpod_flash/cli/commands/init.py`

## flash run

Start local FastAPI development server.

```bash
flash run [--auto-provision] [--host HOST] [--port PORT]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--auto-provision` | off | Pre-deploy all endpoints before serving |
| `--host` | `localhost` | Server host (or `FLASH_HOST` env) |
| `--port` | `8888` | Server port (or `FLASH_PORT` env) |

- Starts FastAPI server with live reload
- API explorer at `http://localhost:8888/docs`
- `--auto-provision` eliminates cold-start delays

**Implementation**: `src/runpod_flash/cli/commands/run.py`

## flash build

Build deployment artifact from source code.

```bash
flash build [--exclude PACKAGES] [--keep-build] [--preview]
```

| Option | Description |
|--------|-------------|
| `--exclude pkg1,pkg2` | Skip packages already in base Docker image |
| `--keep-build` | Don't delete `.flash/.build/` after packaging |
| `--preview` | Build then run in local Docker containers |

**Build steps**:
1. Scan for `@remote` decorators (`RemoteDecoratorScanner`)
2. Group functions by `resource_config`
3. Create `flash_manifest.json` (`ManifestBuilder`)
4. Install dependencies for Linux x86_64
5. Package into `.flash/artifact.tar.gz`

**Output**:
- `.flash/artifact.tar.gz` - Deployment package
- `.flash/flash_manifest.json` - Service discovery config
- `.flash/.build/` - Temp build dir (removed unless `--keep-build`)

**500MB deployment limit** - use `--exclude` for packages in base image:
```bash
flash build --exclude torch,torchvision,torchaudio  # GPU deployments
```

**`--preview` mode**: Creates Docker containers per resource config, starts mothership on `localhost:8000`, enables end-to-end local testing.

**Implementation**: `src/runpod_flash/cli/commands/build.py`

## flash deploy

Deployment environment management.

### flash deploy new

```bash
flash deploy new <env_name> [--app-name NAME]
```

Creates a new deployment environment:
1. Creates FlashApp in RunPod (if first environment)
2. Creates FlashEnvironment
3. Provisions mothership endpoint

### flash deploy send

```bash
flash deploy send <env_name> [--app-name NAME]
```

Deploys built archive to environment:
1. Uploads `.flash/artifact.tar.gz` to S3
2. Provisions all child endpoints upfront
3. Activates environment with archive

**Prerequisite**: `flash build` must have been run first.

### flash deploy list

```bash
flash deploy list [--app-name NAME]
```

Lists all environments showing: name, ID, active build, creation time.

### flash deploy info

```bash
flash deploy info <env_name> [--app-name NAME]
```

Detailed environment info: status, endpoints, network volumes.

### flash deploy delete

```bash
flash deploy delete <env_name> [--app-name NAME]
```

Deletes environment (requires double confirmation). Undeploys associated resources first.

**Implementation**: `src/runpod_flash/cli/commands/deploy.py`

## flash undeploy

Undeploy resources.

```bash
flash undeploy [name|list]
```

- `flash undeploy list` - List all deployed resources
- `flash undeploy <name>` - Undeploy specific resource

**Implementation**: `src/runpod_flash/cli/commands/undeploy.py`

## flash env

Environment variable management.

```bash
flash env list              # List environments
flash env create <name>     # Create environment
flash env info <name>       # Show environment details
flash env delete <name>     # Delete environment
```

**Implementation**: `src/runpod_flash/cli/commands/env.py`

## flash app

Flash app management.

```bash
flash app list              # List all apps
flash app get <name>        # Get app details
```

**Implementation**: `src/runpod_flash/cli/commands/apps.py`

## Development Commands (Makefile)

```bash
make dev              # Install in editable mode with all deps
make test-unit        # Run unit tests (parallel)
make test             # Run all tests (parallel)
make test-serial      # Run all tests (serial, for debugging)
make test-fast        # Fast-fail mode
make lint             # Ruff check
make lint-fix         # Ruff auto-fix
make format           # Ruff format
make format-check     # Check formatting
make index            # Rebuild code intelligence index
make build            # Build PyPI package
make validate-wheel   # Validate wheel packaging
make clean            # Remove build artifacts
```

## Configuration Files

| File | Purpose |
|------|---------|
| `.env` | API keys and local config |
| `.flashignore` | Files to exclude from deployment |
| `.gitignore` | Git ignore patterns |
| `pyproject.toml` | Package config, dependencies, tool settings |
| `flash_manifest.json` | Build-time function-to-endpoint map |
| `.flash/artifact.tar.gz` | Deployment package |
| `.runpod/resources.pkl` | Local resource state (pickled) |
