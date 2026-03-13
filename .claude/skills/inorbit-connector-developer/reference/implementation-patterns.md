# Implementation Patterns

Concrete code patterns from real connectors. Use as templates when implementing each component.

## Configuration Models

### Fleet Connector Config

```python
from pydantic import field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

from inorbit_connector.models import ConnectorConfig, RobotConfig


class MyRobotConfig(RobotConfig):
    """Per-robot configuration."""

    robot_id: str                        # InOrbit robot ID
    fleet_robot_id: str | int            # Target system robot ID


class MyTargetConfig(BaseSettings):
    """Target system connection settings."""

    model_config = SettingsConfigDict(
        env_prefix="INORBIT_MYTARGET_",
        case_sensitive=False,
        env_ignore_empty=True,
    )

    host: str
    port: int = 80
    username: str
    password: str
    use_ssl: bool = False
    verify_ssl: bool = True
    ssl_ca_bundle: str | None = None


class MyConnectorConfig(ConnectorConfig):
    """Top-level connector configuration."""

    connector_config: MyTargetConfig
    fleet: list[MyRobotConfig]

    @field_validator("connector_type")
    @classmethod
    def check_connector_type(cls, v: str) -> str:
        if v != "my_target":
            raise ValueError(f"Expected connector_type 'my_target', got '{v}'")
        return v

    @field_validator("fleet")
    @classmethod
    def validate_unique_fleet_ids(cls, v: list[MyRobotConfig]) -> list[MyRobotConfig]:
        ids = [r.fleet_robot_id for r in v]
        if len(ids) != len(set(ids)):
            raise ValueError("fleet_robot_id values must be unique")
        return v
```

### Single-Robot Config

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

from inorbit_connector.models import ConnectorConfig


class MyRobotTargetConfig(BaseSettings):
    """Target robot connection settings."""

    model_config = SettingsConfigDict(
        env_prefix="INORBIT_MYTARGET_",
        case_sensitive=False,
        env_ignore_empty=True,
    )

    host: str
    port: int = 80
    username: str
    password: str
```

## API Client

```python
import logging

import httpx
from tenacity import (
    before_sleep_log,
    retry,
    retry_if_exception,
    stop_after_attempt,
    wait_exponential_jitter,
)


def _should_retry(exc: BaseException) -> bool:
    """Retry on transient errors only."""
    if isinstance(exc, (httpx.TimeoutException, httpx.ConnectError)):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in (408, 429, 500, 502, 503, 504)
    return False


def _retry_decorator():
    return retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential_jitter(initial=1, max=10),
        retry=retry_if_exception(_should_retry),
        before_sleep=before_sleep_log(logging.getLogger(__name__), logging.WARNING),
    )


class MyAPIClient:
    """Async HTTP client for the target system API."""

    def __init__(
        self,
        host: str,
        port: int,
        username: str,
        password: str,
        use_ssl: bool = False,
        verify_ssl: bool = True,
        ssl_ca_bundle: str | None = None,
    ) -> None:
        protocol = "https" if use_ssl else "http"
        base_url = f"{protocol}://{host}:{port}/api/v1"

        verify: bool | str = verify_ssl
        if verify_ssl and ssl_ca_bundle:
            verify = ssl_ca_bundle

        self._client = httpx.AsyncClient(
            base_url=base_url,
            auth=httpx.BasicAuth(username, password),
            verify=verify,
            timeout=30.0,
        )
        self.logger = logging.getLogger(self.__class__.__name__)

    async def close(self) -> None:
        await self._client.aclose()

    @_retry_decorator()
    async def _get(self, endpoint: str, **kwargs) -> httpx.Response:
        response = await self._client.get(endpoint, **kwargs)
        response.raise_for_status()
        return response

    @_retry_decorator()
    async def _post(self, endpoint: str, **kwargs) -> httpx.Response:
        response = await self._client.post(endpoint, **kwargs)
        response.raise_for_status()
        return response

    @_retry_decorator()
    async def _put(self, endpoint: str, **kwargs) -> httpx.Response:
        response = await self._client.put(endpoint, **kwargs)
        response.raise_for_status()
        return response

    # --- Target-specific methods ---

    async def get_robots(self) -> list[dict]:
        response = await self._get("/robots")
        return response.json()

    async def get_robot(self, robot_id: str | int) -> dict:
        response = await self._get(f"/robots/{robot_id}")
        return response.json()

    async def pause_robot(self, robot_id: str | int) -> None:
        await self._put(f"/robots/{robot_id}/state", json={"state": "paused"})

    async def resume_robot(self, robot_id: str | int) -> None:
        await self._put(f"/robots/{robot_id}/state", json={"state": "ready"})
```

## Data Poller

```python
import asyncio
import logging

logger = logging.getLogger(__name__)


class DataPoller:
    """Background poller that caches robot state for the execution loop."""

    def __init__(
        self,
        client,  # MyAPIClient
        robot_id_to_fleet_id: dict[str, str | int],
        update_freq: float,
    ) -> None:
        self._client = client
        self._robot_id_to_fleet_id = robot_id_to_fleet_id
        self._update_freq = update_freq
        self._robot_data: dict[str, dict] = {}
        self._api_connected: dict[str, bool] = {}
        self._stop_event = asyncio.Event()
        self._tasks: list[asyncio.Task] = []

    def start(self) -> None:
        for robot_id in self._robot_id_to_fleet_id:
            task = asyncio.create_task(self._poll_robot(robot_id))
            self._tasks.append(task)

    async def stop(self) -> None:
        self._stop_event.set()
        if self._tasks:
            done, pending = await asyncio.wait(self._tasks, timeout=2.0)
            for task in pending:
                task.cancel()

    async def _poll_robot(self, robot_id: str) -> None:
        fleet_id = self._robot_id_to_fleet_id[robot_id]
        interval = 1.0 / self._update_freq

        while not self._stop_event.is_set():
            try:
                data = await self._client.get_robot(fleet_id)
                self._robot_data[robot_id] = data
                self._api_connected[robot_id] = True
            except Exception as e:
                logger.error("Error polling %s: %s", robot_id, e)
                self._api_connected[robot_id] = False

            try:
                await asyncio.wait_for(self._stop_event.wait(), timeout=interval)
            except TimeoutError:
                pass

    # --- Data getters (return None if unavailable) ---

    def get_robot_pose(self, robot_id: str) -> dict | None:
        data = self._robot_data.get(robot_id)
        if not data:
            return None
        return {
            "x": data["position"]["x"],
            "y": data["position"]["y"],
            "yaw": data["position"]["orientation"],
            "frame_id": data.get("map_id", "map"),
        }

    def get_robot_key_values(self, robot_id: str) -> dict | None:
        data = self._robot_data.get(robot_id)
        if not data:
            return None
        return {
            "battery": data["battery_percent"] / 100.0,
            "state": data.get("state", "unknown"),
            "api_connected": self._api_connected.get(robot_id, False),
        }
```

## Command Handling

```python
from enum import StrEnum
from typing import Any

from inorbit_connector.commands import (
    CommandFailure,
    CommandModel,
    CommandResultCode,
    ExcludeUnsetMixin,
    parse_custom_command_args,
)


class CustomScripts(StrEnum):
    """Custom command script names. Must match ActionDefinition filename values."""
    PAUSE_ROBOT = "pause_robot"
    RESUME_ROBOT = "resume_robot"
    DOCK = "dock"


class CommandDock(ExcludeUnsetMixin, CommandModel):
    """Validated arguments for the dock command."""
    station_id: str | None = None


# --- In the connector class ---

async def _handle_custom_command(
    self,
    robot_id: str,
    args: list,
    options: dict,
) -> None:
    """Dispatch custom commands."""
    result_function = options["result_function"]
    script_name, script_args = parse_custom_command_args(args)

    fleet_id = self._robot_id_to_fleet_id[robot_id]

    match script_name:
        case CustomScripts.PAUSE_ROBOT:
            await self._api_client.pause_robot(fleet_id)
        case CustomScripts.RESUME_ROBOT:
            await self._api_client.resume_robot(fleet_id)
        case CustomScripts.DOCK:
            cmd = CommandDock.model_validate(script_args)
            await self._api_client.dock(fleet_id, cmd.station_id)
        case _:
            raise CommandFailure(
                execution_status_details=f"Unknown command: {script_name}",
                stderr=f"Command '{script_name}' is not supported",
            )

    result_function(CommandResultCode.SUCCESS)
```

## Main Connector (FleetConnector)

```python
import logging
from typing import override

from inorbit_connector.connector import FleetConnector, CommandFailure
from inorbit_connector.commands import parse_custom_command_args
from inorbit_connector.models import MapConfigTemp

from .api.client import MyAPIClient
from .api.data_poller import DataPoller
from .commands import CustomScripts
from .config.models import MyConnectorConfig

logger = logging.getLogger(__name__)


class MyFleetConnector(FleetConnector):
    """Fleet connector for MyTarget system."""

    def __init__(self, config: MyConnectorConfig) -> None:
        super().__init__(config)
        self._api_client: MyAPIClient | None = None
        self._data_poller: DataPoller | None = None

        # Build bidirectional ID mapping
        self._robot_id_to_fleet_id: dict[str, str | int] = {
            r.robot_id: r.fleet_robot_id for r in config.fleet
        }
        self._fleet_id_to_robot_id: dict[str | int, str] = {
            v: k for k, v in self._robot_id_to_fleet_id.items()
        }

    @property
    def config(self) -> MyConnectorConfig:
        return self._config

    @override
    async def _connect(self) -> None:
        cfg = self.config.connector_config
        self._api_client = MyAPIClient(
            host=cfg.host,
            port=cfg.port,
            username=cfg.username,
            password=cfg.password,
            use_ssl=cfg.use_ssl,
            verify_ssl=cfg.verify_ssl,
        )
        self._data_poller = DataPoller(
            client=self._api_client,
            robot_id_to_fleet_id=self._robot_id_to_fleet_id,
            update_freq=self.config.update_freq,
        )
        self._data_poller.start()

    @override
    async def _disconnect(self) -> None:
        if self._data_poller:
            await self._data_poller.stop()
        if self._api_client:
            await self._api_client.close()

    @override
    async def _execution_loop(self) -> None:
        for robot_id in self.robot_ids:
            if pose := self._data_poller.get_robot_pose(robot_id):
                self.publish_robot_pose(robot_id, **pose)

            if kv := self._data_poller.get_robot_key_values(robot_id):
                self.publish_robot_key_values(robot_id, **kv)

    @override
    async def _inorbit_robot_command_handler(
        self, robot_id: str, command_name: str, args: list, options: dict
    ) -> None:
        if command_name == "customCommand":
            await self._handle_custom_command(robot_id, args, options)

    @override
    def _is_fleet_robot_online(self, robot_id: str) -> bool:
        if self._data_poller:
            return self._data_poller._api_connected.get(robot_id, False)
        return False

    async def _handle_custom_command(
        self, robot_id: str, args: list, options: dict
    ) -> None:
        # See Command Handling section above
        ...
```

## Main Connector (Connector — Single-Robot)

```python
from typing import override

from inorbit_connector.connector import Connector

from .api.client import MyAPIClient
from .config.models import MyConnectorConfig


class MyRobotConnector(Connector):
    """Single-robot connector for MyTarget."""

    def __init__(self, robot_id: str, config: MyConnectorConfig) -> None:
        super().__init__(robot_id, config)
        self._api_client: MyAPIClient | None = None

    @override
    async def _connect(self) -> None:
        cfg = self._config.connector_config
        self._api_client = MyAPIClient(host=cfg.host, port=cfg.port, ...)

    @override
    async def _disconnect(self) -> None:
        if self._api_client:
            await self._api_client.close()

    @override
    async def _execution_loop(self) -> None:
        data = await self._api_client.get_robot(self.robot_id)
        self.publish_pose(x=data["x"], y=data["y"], yaw=data["yaw"], frame_id="map")
        self.publish_key_values(battery=data["battery"] / 100.0)

    @override
    async def _inorbit_command_handler(
        self, command_name: str, args: list, options: dict
    ) -> None:
        if command_name == "customCommand":
            await self._handle_custom_command(args, options)

    @override
    def _is_robot_online(self) -> bool:
        return self._api_client is not None
```

## Entry Point

```python
import logging
import signal
import sys

from inorbit_connector.utils import read_yaml

from .config.models import MyConnectorConfig
from .src.connector import MyFleetConnector

LOGGER = logging.getLogger(__name__)


def start() -> None:
    """Main entry point."""
    import argparse

    parser = argparse.ArgumentParser(description="MyTarget InOrbit Connector")
    parser.add_argument("-c", "--config", type=str, required=True,
                        help="Path to YAML configuration file")
    args = parser.parse_args()

    try:
        yaml_data = read_yaml(args.config)
        config = MyConnectorConfig(**yaml_data)
    except FileNotFoundError:
        LOGGER.error("Configuration file '%s' not found", args.config)
        sys.exit(1)
    except ValueError as e:
        LOGGER.error("Configuration error: %s", e)
        sys.exit(1)

    connector = MyFleetConnector(config)
    connector.start()

    # Suppress noisy loggers
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("httpcore").setLevel(logging.INFO)

    signal.signal(signal.SIGINT, lambda sig, frame: connector.stop())
    connector.join()
```

## Testing

### conftest.py

```python
import asyncio

import pytest


@pytest.fixture(autouse=True)
def _fast_asyncio_sleep(monkeypatch):
    """Short-circuit asyncio.sleep to keep tests fast."""
    original_sleep = asyncio.sleep

    async def _sleep_stub(delay, *args, **kwargs):
        await original_sleep(0)

    monkeypatch.setattr(asyncio, "sleep", _sleep_stub)
    yield
```

### Config Model Tests

```python
import copy

import pytest

from my_connector.src.config.models import MyConnectorConfig


@pytest.fixture()
def base_config_data() -> dict:
    return {
        "connector_type": "my_target",
        "connector_config": {
            "host": "192.168.1.50",
            "port": 80,
            "username": "user",
            "password": "pass",
        },
        "fleet": [
            {"robot_id": "robot-1", "fleet_robot_id": 1},
            {"robot_id": "robot-2", "fleet_robot_id": 2},
        ],
    }


def test_valid_config(base_config_data: dict) -> None:
    config = MyConnectorConfig(**base_config_data)
    assert config.connector_type == "my_target"
    assert len(config.fleet) == 2


def test_duplicate_fleet_ids_rejected(base_config_data: dict) -> None:
    data = copy.deepcopy(base_config_data)
    data["fleet"][1]["fleet_robot_id"] = data["fleet"][0]["fleet_robot_id"]
    with pytest.raises(ValueError, match="unique"):
        MyConnectorConfig(**data)
```

### Command Handler Tests

```python
import logging
from types import SimpleNamespace
from unittest.mock import AsyncMock

import pytest

from inorbit_connector.commands import CommandResultCode

from my_connector.src.connector import MyFleetConnector


def _make_connector():
    """Create a connector with mock dependencies."""
    connector = MyFleetConnector.__new__(MyFleetConnector)
    connector._api_client = SimpleNamespace(
        pause_robot=AsyncMock(),
        resume_robot=AsyncMock(),
    )
    connector._robot_id_to_fleet_id = {"robot-1": 101}
    connector._logger = logging.getLogger("test")
    return connector


def _make_result_function():
    calls = []
    def _result(code, **kwargs):
        calls.append((code, kwargs))
    return calls, _result


@pytest.mark.asyncio
async def test_pause_command() -> None:
    connector = _make_connector()
    calls, result_fn = _make_result_function()

    await connector._handle_custom_command(
        robot_id="robot-1",
        args=[{"filename": "pause_robot"}],
        options={"result_function": result_fn},
    )

    connector._api_client.pause_robot.assert_awaited_once_with(101)
    assert calls[0][0] == CommandResultCode.SUCCESS
```

### HTTP Client Tests (with pytest-httpx)

```python
import pytest
from pytest_httpx import HTTPXMock

from my_connector.src.api.client import MyAPIClient


@pytest.fixture
async def client():
    c = MyAPIClient(host="localhost", port=8080, username="u", password="p")
    yield c
    await c.close()


@pytest.mark.asyncio
async def test_get_robots(client: MyAPIClient, httpx_mock: HTTPXMock) -> None:
    httpx_mock.add_response(
        url="http://localhost:8080/api/v1/robots",
        json=[{"id": 1, "name": "robot-1"}],
    )
    robots = await client.get_robots()
    assert len(robots) == 1
    assert robots[0]["id"] == 1
```

## Mock API Client

```python
import random


class MockAPIClient:
    """Mock client for development without hardware."""

    def __init__(self) -> None:
        self._robots = {
            1: {"id": 1, "position": {"x": 0.0, "y": 0.0, "orientation": 0.0},
                "battery_percent": 85.0, "state": "ready"},
            2: {"id": 2, "position": {"x": 1.0, "y": 2.0, "orientation": 1.57},
                "battery_percent": 72.0, "state": "executing"},
        }

    async def close(self) -> None:
        pass

    async def get_robot(self, robot_id: int) -> dict:
        robot = self._robots[robot_id]
        # Simulate movement
        robot["position"]["x"] += random.uniform(-0.1, 0.1)
        robot["position"]["y"] += random.uniform(-0.1, 0.1)
        robot["battery_percent"] = max(0, robot["battery_percent"] - 0.01)
        return robot

    async def pause_robot(self, robot_id: int) -> None:
        self._robots[robot_id]["state"] = "paused"

    async def resume_robot(self, robot_id: int) -> None:
        self._robots[robot_id]["state"] = "ready"
```

## Example Config YAML

```yaml
# config/fleet.example.yaml
location_tz: America/Los_Angeles

logging:
  log_level: INFO

connector_type: my_target
update_freq: 1.0

connector_config:
  host: 192.168.1.50
  port: 80
  username: admin
  password: changeme
  use_ssl: false
  verify_ssl: true

fleet:
  - robot_id: my-target-robot-1
    fleet_robot_id: 1
  - robot_id: my-target-robot-2
    fleet_robot_id: 2
  - robot_id: my-target-robot-3
    fleet_robot_id: 3

# Optional: static maps
# maps:
#   map_frame:
#     file: ./config/map.png
#     map_id: map
#     map_label: Facility Map
#     origin_x: 0.0
#     origin_y: 0.0
#     resolution: 0.05
```
