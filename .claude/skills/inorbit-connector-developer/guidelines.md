# Coding Standards & Quality Guidelines

## Repository Standards

- **License**: MIT with REUSE compliance (`REUSE.toml` + `LICENSES/MIT.txt`)
- **Linting**: ruff with `line-length = 100`
- **Python**: Specify correct `requires-python` in pyproject.toml. The cookiecutter supports
  3.10–3.13 (`>=3.10,<3.14`) and defaults the generated project to 3.13.
- **Dependencies**: Use compatible version ranges (`~=` for minor pins, `>=` for minimum)

## Import Order

1. Standard library
2. Third-party packages (`httpx`, `tenacity`, `pydantic`, etc.)
3. InOrbit framework (`inorbit_connector`, `inorbit_edge`)
4. Local modules

```python
import asyncio
import logging
from enum import StrEnum

import httpx
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

from inorbit_connector.connector import FleetConnector
from inorbit_connector.commands import CommandFailure, parse_custom_command_args

from .config.models import MyConnectorConfig
```

## Type Annotations

Use Python 3.10+ union syntax. **Never import `Optional`, `Dict`, `List`, or `Tuple` from `typing`**:
- `str | None` not `Optional[str]` — applies to all Pydantic model fields too
- `dict[str, Any]` not `Dict[str, Any]`
- `list[str]` not `List[str]`
- `tuple[str, int]` not `Tuple[str, int]`

## Override Decorator

Always use `@override` on methods that override a parent class:

```python
from typing import override

class MyConnector(FleetConnector):
    @override
    async def _connect(self) -> None:
        ...

    @override
    async def _execution_loop(self) -> None:
        ...
```

## Docstrings

Google style:

```python
def method(self, arg: str) -> dict[str, Any]:
    """Brief summary.

    Args:
        arg: Description.

    Returns:
        Description of return value.

    Raises:
        ValueError: When invalid.
    """
```

## Logging

- **In connector classes**: Use `self._logger` (inherited from FleetConnector/Connector)
- **In other modules**: `logger = logging.getLogger(__name__)`
- **In API client classes**: `self.logger = logging.getLogger(self.__class__.__name__)`

## Enums

Use `StrEnum` for string constants (avoids magic strings):

```python
from enum import StrEnum

class CustomScripts(StrEnum):
    PAUSE_ROBOT = "pause_robot"
    RESUME_ROBOT = "resume_robot"
```

## Metrics

See [reference/metrics.md](reference/metrics.md) for the full guide. Critical rules:

- **Get a meter via `get_connector_meter(CONNECTOR_TYPE)`**, never via `inorbit_edge.metrics.get_meter(...)` in vendor code.
- **Drop the vendor prefix from instrument names** — the wrapper adds `<connector_type>.` automatically. Write `meter.create_counter("mission.failures")`, not `meter.create_counter("acme.mission.failures")`.
- **Record every HTTP call through the framework helpers**: `record_upstream_http_request()` on success, `record_upstream_http_error()` on failure (timeout, connect failure, non-2xx). Both live in `inorbit_connector.metrics.http`. Don't roll your own request/error counters.
- **Always normalize the `endpoint` attribute** via `EndpointMapper` (preferred) or `PathTemplater` before passing to either helper. Raw paths containing IDs/UUIDs irreversibly pollute the metric descriptor on Stackdriver.
- **Never declare your own `MeterProvider` / `PrometheusMetricReader` / `start_http_server`.** The framework wires the provider once when `MetricsConfig.enabled=True`.
- **Bounded attribute values only** — enums, classification buckets, IDs from a known small set. No exception messages, raw URLs, or user input.
- **Size histogram buckets explicitly.** OTEL's default 15-boundary histogram emits 18 values per scrape per attribute combination — ~18× the ingestion cost of a counter. Pass `explicit_bucket_boundaries=(0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0)` (or whatever fits your latency range) and keep histograms to ≤2 attribute dimensions.

## Testing

- **Framework**: pytest with pytest-asyncio for async tests
- **HTTP mocking**: pytest-httpx (`HTTPXMock` fixture)
- **Coverage**: Sensible coverage for business logic
- **Patterns**:
  - `@pytest.mark.asyncio` for async test functions
  - Monkeypatch `asyncio.sleep` to keep tests fast
  - Mock external dependencies (API clients, robot sessions)
  - Test both success and error paths

## CI/CD

- **Test runner**: tox (`uv run tox`)
- **Linting**: ruff (`uv run ruff check`)
- **REUSE compliance**: checked in CI
- **Docker builds**: triggered on version bumps
- **Versioning**: bump-my-version (`uv run bump-my-version bump <part>`)

## Docker

- Multi-stage builds (build stage + runtime stage)
- Python 3.13 base image
- Environment variable support for configuration
- docker-compose example with volume mounts

## Development Workflow

- PRs required for all changes to main
- Tests must pass before merging
- Linter must pass
- Mock API client class recommended for development without hardware
