# Coding Standards & Quality Guidelines

## Repository Standards

- **License**: MIT with REUSE compliance (`REUSE.toml` + `LICENSES/MIT.txt`)
- **Linting**: ruff with `line-length = 100`
- **Python**: Specify correct `requires-python` in pyproject.toml (typically `>=3.13`)
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
