---
name: inorbit-connector-developer
description: Build or improve InOrbit robot connectors. Use when creating a new connector from an API spec, improving an existing connector, or asking about InOrbit connector architecture. Covers the full lifecycle from API analysis through implementation, testing, CaC configuration, and optionally edge mission execution.
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

**Reference connectors:**

> For **v3 config patterns** (`ConnectorRootConfig` / `ConnectorSpecificConfig` / `CONNECTOR_TYPE`),
> the source of truth is the cookiecutter-generated project (`src/config/models.py`,
> `src/connector.py`, `src/commands.py`) plus this skill's reference docs. The reference connectors
> below predate v3 — use them for architecture and API integration, but translate any
> `ConnectorConfig` / hand-written `env_prefix` / `account_id` patterns to the v3 form (see the
> "Do Not" table).

- `github.com/inorbit-ai/inorbit-robot-connectors/omron_flowcore_connector` — **recommended code
  reference** for structure. Correct `parse_custom_command_args` + `@override` patterns and edge
  executor v4 integration throughout. Study it for architecture, command handling, and edge
  mission integration — but translate its config models to the v3 pattern.
- `github.com/inorbit-ai/inorbit-robot-connectors/mir_connector` — useful if building a MiR
  connector. Mostly modern but has accumulated some older patterns due to its age; use for
  MiR API structure, not as a code pattern reference.

## Do Not

These patterns appear in older connectors and in training data. Never use:

| Wrong | Right | Why |
|-------|-------|-----|
| `dict(zip(args[::2], args[1::2]))` | `parse_custom_command_args(args)` | Framework handles edge cases and validation |
| `Optional[str]` | `str \| None` | Python 3.10+ union syntax, project standard |
| `battery=raw_value` (0-100) | `battery=raw_value / 100.0` | InOrbit expects 0-1 float |
| `# @override` (comment) | `@override` (import + decorator) | Static analysis can't enforce commented decorators |
| `api_key: ""` in YAML | Omit field or use example value | Empty strings override env vars in pydantic-settings |
| `ConnectorConfig` (BaseModel) | `ConnectorRootConfig[T]` + `ConnectorSpecificConfig` | v3 config is pydantic-settings; old class removed |
| `SettingsConfigDict(env_prefix=...)` on vendor config | set `CONNECTOR_TYPE` class var | v3 derives the `INORBIT_{TYPE}_` prefix automatically |
| `account_id` config field | (removed) | v3 resolves the account ID automatically via the edge SDK |
| `"customCommand"` string literal | `COMMAND_CUSTOM_COMMAND` from `inorbit_edge.robot` | Use the constant, not a magic string |
| `inorbit_edge.metrics.get_meter("inorbit_acme_connector")` in vendor code | `inorbit_connector.metrics.get_connector_meter("acme")` | Wrapper adds the vendor prefix; CI lint flags direct `get_meter` |
| `meter.create_counter("acme.api.errors", ...)` after `get_connector_meter("acme")` | `meter.create_counter("api.errors", ...)` | Wrapper adds the `acme.` prefix once; doubled prefixes are ugly and easy to drift |
| Manual `MeterProvider` + `PrometheusMetricReader` + `start_http_server` in connector | `MetricsConfig` in YAML | Framework owns the provider; manual setup causes namespace drift |
| `record_upstream_http_request(..., endpoint=raw_path)` | `endpoint=EndpointMapper(...)(raw_path)` | Raw paths with IDs pollute the metric descriptor irreversibly on Stackdriver |
| `record_upstream_http_request` on a 4xx/5xx response | `record_upstream_http_error(..., error_kind="http_4xx"/"http_5xx")` | Successes and errors are separate metrics; mixing them blurs alerting |
| `meter.create_histogram("api.duration")` with default buckets and 3+ attributes | Pass `explicit_bucket_boundaries=(0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0)` and drop one attribute | Default OTEL has 16 buckets → 18× the ingestion of a counter. With 3 attribute dimensions that adds up to ~500MB/month per histogram. See [reference/metrics.md](reference/metrics.md) |
| `attributes={"error_message": str(e)}` | `attributes={"error_kind": type(e).__name__}` | Exception strings are unbounded; bucket by class instead |

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

> **Battery**: InOrbit expects a 0-1 float. Fleet APIs almost universally return 0-100. Always
> divide by 100.0 at publish time. Publishing raw 0-100 values silently breaks the battery widget.

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
| inorbit-connector | ~=3.0.0 | 3.0.x | ? |
| pytest | ~=8.4 | 8.5.2 | ? |
| ... | ... | ... | ... |

> The generated `pyproject.toml` pins `inorbit-connector[system-stats]~=3.0.0`, which pulls in
> `inorbit-edge>=3.0,<4.0` and `pydantic-settings` transitively — those are not listed directly.

**Do NOT update `pyproject.toml` until the user reviews the table and confirms which versions to upgrade.** InOrbit packages in particular may have breaking changes.

**After user approval**, update `pyproject.toml`, then lock, install, and verify:
```bash
uv lock && uv sync --extra=dev && uv run pytest && uv run ruff check
```

### Phase 6: Implementation

Read [reference/framework-api.md](reference/framework-api.md) and [reference/implementation-patterns.md](reference/implementation-patterns.md) for API surface and code examples.

Implement in this order:
1. **Configuration models** (`src/config/models.py`) — `ConnectorSpecificConfig` subclass (vendor fields, with `CONNECTOR_TYPE`), `RobotConfig` subclass (per-robot fields), and a `ConnectorRootConfig[...]` parametrization (top level + domain validators). The cookiecutter prefills this file.
2. **API client** (`src/api/client.py`) — httpx.AsyncClient, retry, auth, SSL. **Record every request through `record_upstream_http_request()` / `record_upstream_http_error()`** from `inorbit_connector.metrics.http` — see [reference/metrics.md](reference/metrics.md).
3. **Data poller** (`src/api/data_poller.py`) — background polling, local cache, getters
4. **Command models** (`src/commands.py`) — StrEnum scripts, CommandModel validation
5. **Main connector** (`src/connector.py`) — FleetConnector or Connector subclass with all overrides
6. **Domain metrics** (`src/metrics.py`) — `get_connector_meter(CONNECTOR_TYPE)` + module-level instruments for vendor-specific signals worth alerting on. See [reference/metrics.md](reference/metrics.md). Skip this file if you have no domain instruments to add (framework + canonical HTTP cover most needs).
7. **Entry point** — YAML config loading, signal handling

### Phase 7: CaC Configuration

CaC is not an afterthought. Write each ActionDefinition when you implement its command, and
each DataSourceDefinition when you add a key-value. The `cac/` skeleton in the generated project
has placeholder files — fill them in as you go, not after implementation is complete.

Read [reference/cac-objects.md](reference/cac-objects.md) for InOrbit Configuration-as-Code reference.

Fill in `cac/` YAML files for:
- ActionDefinition — custom commands exposed in InOrbit UI
- DataSourceDefinition — key-value and derived data sources
- MissionTracking — mission tracking configuration
- RobotFootprint — robot body and buffer polygons
- Preferences — UI panel settings

**If edge mission execution is needed**: Also read [reference/mission-execution.md](reference/mission-execution.md) and ensure:
- executeMission, cancelMission, updateMission actions are defined (account scope)
- MissionTracking includes `execution.executor.type: edge`
- MissionDefinition objects define available missions

### Phase 8: Testing

Read [reference/implementation-patterns.md#testing](reference/implementation-patterns.md) for test patterns.

Key points:
- pytest-asyncio for async tests
- pytest-httpx for HTTP mocking
- Test config validation, API client, data transforms, command handling
- Run: `uv run pytest && uv run ruff check`

### Phase 9: Documentation

Update `README.md` with:
1. Overview — what target system this connects
2. Prerequisites — target system version, required credentials
3. Installation — how to install
4. Configuration — all config options with examples
5. Usage — how to run
6. Commands — available custom commands
7. Troubleshooting — common issues

## Coding Conventions

See [guidelines.md](guidelines.md) for full details. Key points:

- **Imports**: stdlib -> third-party -> inorbit framework -> local
- **Types**: Python 3.10+ syntax (`str | None`, `dict[str, Any]`)
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
- [ ] Every HTTP call goes through `record_upstream_http_request()` or `record_upstream_http_error()` with a normalized `endpoint` (see [reference/metrics.md](reference/metrics.md))
- [ ] Any vendor-specific domain instruments use `get_connector_meter(CONNECTOR_TYPE)` — never `inorbit_edge.metrics.get_meter` directly
- [ ] No raw paths, exception messages, or unbounded values passed as metric attributes
