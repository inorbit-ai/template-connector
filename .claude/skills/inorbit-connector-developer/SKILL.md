---
name: inorbit-connector-developer
description: Build or improve InOrbit robot connectors. Use when creating a new connector from an API spec, improving an existing connector, or asking about InOrbit connector architecture. Covers the full lifecycle from API analysis through implementation, testing, CaC configuration, and optionally edge mission execution.
disable-model-invocation: true
license: MIT
author: InOrbit, Inc.
version: 0.1.0
---

# InOrbit Connector Development

Guide for building new connectors or improving existing ones.

## Architecture

```
your-connector (new project)
    └── inorbit-connector-python (framework: FleetConnector / Connector)
            └── inorbit-edge (low-level SDK: MQTT + InOrbit cloud)
                    └── [optional] inorbit_edge_executor (edge mission execution)
```

**GitHub repositories for source reference:**
- Framework: `github.com/inorbit-ai/inorbit-connector-python`
- Low-level SDK: `github.com/inorbit-ai/edge-sdk-python`
- Edge missions executor (optional): `github.com/inorbit-ai/inorbit_edge_executor`
- Project generator: `github.com/inorbit-ai/inorbit-connector-cookiecutter`

**Reference connectors (study for patterns):**
- `github.com/inorbit-ai/inorbit-robot-connectors` — collection of single-robot connectors (some with edge missions, some without)

## Entry Points

### New Connector
Follow all phases below in order.

### Existing Connector
1. Read the codebase to understand current state
2. Identify gaps (missing data publishing, commands, CaC, tests, docs)
3. Jump to the relevant phase(s) below

## Phased Workflow

### Phase 1: API Analysis

When provided a robot/fleet API specification (OpenAPI, docs, or URL), analyze it to identify:

**Data endpoints:**
- Robot discovery (list robots, get robot by ID)
- Robot state (pose, battery, status, errors, velocity)
- Maps/environments (if available)
- System stats (CPU, RAM, disk)

**Control endpoints:**
- Navigation commands
- State control (pause, resume, dock)
- Mission/task/job management (if supported)
- Messages/announcements

**Connection details:**
- Authentication method (API key, basic auth, OAuth, bearer token)
- Base URL structure, API versioning
- SSL/TLS requirements
- Rate limits, pagination
- WebSocket/streaming (if available)

**Map to InOrbit concepts:**
| InOrbit Concept | Description | Typical Source |
|-----------------|-------------|----------------|
| Pose | x, y, yaw in a coordinate frame | Position endpoint |
| Odometry | linear/angular speed and distance | Velocity endpoint or derived |
| Key-values | Battery (0-1), state, errors, custom | Status endpoints |
| System stats | CPU %, RAM %, disk % | System endpoint (if available) |
| Maps | Image + origin + resolution + frame_id | Map endpoint |

### Phase 2: PRD Generation

**Write a PRD file to disk** at `.prd/PRD.md` (dot-prefixed directory so the cookiecutter won't overwrite it). This file is a key artifact — the user will review and edit it before implementation begins.

The PRD must be **exhaustive**. Cover every data source and command the API supports, not just a representative subset. The goal is to publish **all relevant data** and implement **all useful actions**.

Contents:
1. Overview (target system, integration type: fleet or single-robot, API version)
2. Authentication & connection strategy
3. Data publishing — **list every data point** that can be reported:
   - Pose (source endpoint, coordinate system, units)
   - Odometry (source or derivation method)
   - Key-values: **one row per key** (battery, state, errors, mode, mission status, firmware version, network status, etc. — include everything available from the API)
   - System stats (CPU, RAM, disk — if available)
   - Maps (format, metadata)
4. Command handling — **list every command** that can be implemented:
   - Navigation goals
   - State control (pause, resume, dock, undock, etc.)
   - Mission/task management (queue, abort, clear queue, etc.)
   - Configuration commands (localize, set mode, etc.)
   - **Include one row per command** with: command name, API endpoint, parameters, notes
5. Configuration schema (connector-level, per-robot)
6. Error handling and retry strategy
7. Testing strategy
8. Open questions

### Phase 3: Feedback Loop

After writing the PRD to disk, tell the user the file path and ask them to review it. Ask specifically:
1. Are the identified endpoints and data mappings correct?
2. **Are there any data sources or commands missing?** (The PRD should be exhaustive — flag anything from the API that was intentionally excluded and explain why.)
3. **Does this connector need edge mission execution?** (i.e., should missions be dispatched and executed locally via custom behavior trees, or using the default behavior available through cloud missions execution?)
4. Any special authentication or SSL requirements?
5. What is the target directory for the project?

The answer to question 3 determines:
- Whether [reference/mission-execution.md](reference/mission-execution.md) is relevant
- Which CaC objects to configure (executeMission/cancelMission/updateMission actions, MissionTracking with edge executor, MissionDefinition)

**Update the PRD file on disk** based on feedback before proceeding. The user may also edit the file directly — re-read it before continuing.

### Phase 4: Project Generation

**Generate using cookiecutter:**
```bash
pipx run cookiecutter gh:inorbit-ai/inorbit-connector-cookiecutter
```

Guide the user through the interactive prompts:
- `connector_target`: Human-readable name (e.g., "Acme Fleet Manager")
- `use_current_directory`: `y` if generating inside an existing repo (e.g., template-connector clone)

### Phase 5: Post-Generation Setup

**Critical**: Read the generated `README.md` and follow the instructions at the top (replacements, TODOs).

**Check for dependency updates** by running `uv pip index versions` for each dependency in `pyproject.toml` (both `[project.dependencies]` and `[project.optional-dependencies.dev]`). Then present a summary table like:

| Package | Pinned Version | Latest Available | Update? |
|---------|---------------|-----------------|---------|
| inorbit-connector | ~=2.2.0 | 2.3.1 | ? |
| inorbit-edge | ... | ... | ? |
| pytest | ~=8.4 | 8.5.2 | ? |
| ... | ... | ... | ... |

**Do NOT update `pyproject.toml` until the user reviews the table and confirms which versions to upgrade.** InOrbit packages in particular may have breaking changes.

**After user approval**, update `pyproject.toml`, then lock, install, and verify:
```bash
uv lock && uv sync --extra=dev && uv run pytest && uv run ruff check
```

### Phase 6: Implementation

Read [reference/framework-api.md](reference/framework-api.md) and [reference/implementation-patterns.md](reference/implementation-patterns.md) for API surface and code examples.

Implement in this order:
1. **Configuration models** (`src/config/models.py`) — target-specific fields, per-robot config, validators, env prefix
2. **API client** (`src/api/client.py`) — httpx.AsyncClient, retry, auth, SSL
3. **Data poller** (`src/api/data_poller.py`) — background polling, local cache, getters
4. **Command models** (`src/commands.py`) — StrEnum scripts, CommandModel validation
5. **Main connector** (`src/connector.py`) — FleetConnector or Connector subclass with all overrides
6. **Entry point** — YAML config loading, signal handling

### Phase 7: Testing

Read [reference/implementation-patterns.md#testing](reference/implementation-patterns.md) for test patterns.

Key points:
- pytest-asyncio for async tests
- pytest-httpx for HTTP mocking
- Test config validation, API client, data transforms, command handling
- Run: `uv run pytest && uv run ruff check`

### Phase 8: Documentation

Update `README.md` with:
1. Overview — what target system this connects
2. Prerequisites — target system version, required credentials
3. Installation — how to install
4. Configuration — all config options with examples
5. Usage — how to run
6. Commands — available custom commands
7. Troubleshooting — common issues

### Phase 9: CaC Configuration

Read [reference/cac-objects.md](reference/cac-objects.md) for InOrbit Configuration-as-Code reference.

Create `cac/` directory with YAML files for:
- ActionDefinition — custom commands exposed in InOrbit UI
- DataSourceDefinition — key-value and derived data sources
- MissionTracking — mission tracking configuration
- RobotFootprint — robot body and buffer polygons
- Preferences — UI panel settings

**If edge mission execution is needed**: Also read [reference/mission-execution.md](reference/mission-execution.md) and ensure:
- executeMission, cancelMission, updateMission actions are defined (account scope)
- MissionTracking includes `execution.executor.type: edge`
- MissionDefinition objects define available missions

## Coding Conventions

See [guidelines.md](guidelines.md) for full details. Key points:

- **Imports**: stdlib -> third-party -> inorbit framework -> local
- **Types**: Python 3.13+ syntax (`str | None`, `dict[str, Any]`)
- **Override**: `@override` decorator on all overridden methods
- **Logging**: `self._logger` (inherited) in connector, `logger = logging.getLogger(__name__)` elsewhere
- **Enums**: `StrEnum` for string constants
- **Commands**: `CommandFailure` for errors, `parse_custom_command_args()` for parsing
- **Async**: `httpx.AsyncClient`, `asyncio.Event()` for shutdown, close clients in `_disconnect()`
- **Docstrings**: Google style

## Quality Checklist

- [ ] All abstract methods implemented with `@override`
- [ ] Working test suite (`uv run pytest`)
- [ ] Linter passing (`uv run ruff check`)
- [ ] Configuration examples complete (`fleet.example.yaml`, `example.env`)
- [ ] CaC objects defined in `cac/` directory
- [ ] README documentation complete
- [ ] Docker packaging working
- [ ] Dependency versions specified with compatible ranges (`~=`)
