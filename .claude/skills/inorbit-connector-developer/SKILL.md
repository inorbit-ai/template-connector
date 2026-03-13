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

Create a Product Requirements Document covering:
1. Overview (target system, integration type: fleet or single-robot, API version)
2. Authentication & connection strategy
3. Data publishing (pose, odometry, key-values, system stats, maps)
4. Command handling (navigation goals, custom commands, missions)
5. Configuration schema (connector-level, per-robot)
6. Error handling and retry strategy
7. Testing strategy
8. Open questions

### Phase 3: Feedback Loop

After generating the PRD, ask the user:
1. Are the identified endpoints and data mappings correct?
2. Are there additional custom commands needed?
3. What specific key-values should be tracked?
4. **Does this connector need edge mission execution?** (i.e., should missions be dispatched and executed locally via custom behavior trees, or using the default behavior available through cloud missions execution?)
5. Any special authentication or SSL requirements?
6. What is the target directory for the project?

The answer to question 4 determines:
- Whether `reference/mission-execution.md` is relevant
- Which CaC objects to configure (executeMission/cancelMission/updateMission actions, MissionTracking with edge executor, MissionDefinition)

Update the PRD based on feedback before proceeding.

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

**Check for dependency updates:**
```bash
uv pip index versions inorbit-connector
uv pip index versions inorbit-edge
uv pip index versions inorbit-edge-executor  # only if edge missions needed
```

**If InOrbit dependency updates are available**: Present findings and ask the user before updating. These packages may have breaking changes.

**Lock, install, and verify:**
```bash
uv lock && uv sync --extra=dev && uv run pytest && uv run ruff check
```

### Phase 6: Implementation

Read `reference/framework-api.md` and `reference/implementation-patterns.md` for API surface and code examples.

Implement in this order:
1. **Configuration models** (`src/config/models.py`) — target-specific fields, per-robot config, validators, env prefix
2. **API client** (`src/api/client.py`) — httpx.AsyncClient, retry, auth, SSL
3. **Data poller** (`src/api/data_poller.py`) — background polling, local cache, getters
4. **Command models** (`src/commands.py`) — StrEnum scripts, CommandModel validation
5. **Main connector** (`src/connector.py`) — FleetConnector or Connector subclass with all overrides
6. **Entry point** — YAML config loading, signal handling

### Phase 7: Testing

Read `reference/implementation-patterns.md#testing` for test patterns.

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

Read `reference/cac-objects.md` for InOrbit Configuration-as-Code reference.

Create `cac/` directory with YAML files for:
- ActionDefinition — custom commands exposed in InOrbit UI
- DataSourceDefinition — key-value and derived data sources
- MissionTracking — mission tracking configuration
- RobotFootprint — robot body and buffer polygons
- Preferences — UI panel settings

**If edge mission execution is needed**: Also read `reference/mission-execution.md` and ensure:
- executeMission, cancelMission, updateMission actions are defined (account scope)
- MissionTracking includes `execution.executor.type: edge`
- MissionDefinition objects define available missions

## Coding Conventions

See `guidelines.md` for full details. Key points:

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
