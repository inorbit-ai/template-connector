# Framework API Reference

Based on `inorbit-connector-python` v3.0.0. For full source, see `github.com/inorbit-ai/inorbit-connector-python`.

## FleetConnector

```python
from inorbit_connector.connector import FleetConnector
```

Base class for connectors managing multiple robots through a fleet API.

### Constructor

```python
def __init__(self, config: ConnectorRootConfig, **kwargs) -> None:
```

**Keyword arguments:**
- `register_user_scripts` (bool): Default `False`
- `default_user_scripts_dir` (str): Default `"~/.inorbit_connectors/connector-{ClassName}/local/"`
- `create_user_scripts_dir` (bool): Default `False`
- `register_custom_command_handler` (bool): Default `True`
- `publish_connector_system_stats` (bool): Default `False` (requires the
  `inorbit-connector[system-stats]` extra / `psutil`)

### Properties

- `robot_ids -> list[str]`: Robot IDs from configuration
- `_config`: The `ConnectorRootConfig` object
- `_logger`: Logger instance (inherited, use for logging)

### Abstract Methods (must override)

```python
@abstractmethod
async def _connect(self) -> None:
    """Connect to external services. Called on startup."""

@abstractmethod
async def _disconnect(self) -> None:
    """Disconnect from external services. Called on shutdown."""

@abstractmethod
async def _execution_loop(self) -> None:
    """Main loop iteration. Publish robot data here. Called at update_freq."""

@abstractmethod
async def _inorbit_robot_command_handler(
    self, robot_id: str, command_name: str, args: list, options: dict
) -> None:
    """Handle commands for a specific robot.

    Args:
        robot_id: InOrbit robot ID.
        command_name: Command type (e.g., "customCommand").
        args: Command arguments.
        options: Contains 'result_function' callback.
    """
```

### Data Publishing

```python
def publish_robot_pose(
    self, robot_id: str, x: float, y: float, yaw: float,
    frame_id: str = None, **kwargs
) -> None:

def publish_robot_odometry(self, robot_id: str, **kwargs) -> None:
    # kwargs: linear_speed, angular_speed, ...

def publish_robot_key_values(self, robot_id: str, **kwargs) -> None:
    # kwargs: key=value pairs (battery=0.85, state="ready", ...)

def publish_robot_system_stats(self, robot_id: str, **kwargs) -> None:
    # kwargs: cpu, ram, disk, ...

def publish_robot_map(
    self, robot_id: str, frame_id: str, is_update: bool = False
) -> None:
```

### Map Fetching

```python
async def fetch_robot_map(
    self, robot_id: str, frame_id: str
) -> MapConfigTemp | None:
    """Override to fetch map image from target system.

    Return MapConfigTemp with image bytes, or None if unavailable.
    Called automatically when publish_robot_map() needs a map.
    """
```

### Fleet Management

```python
def update_fleet(self, fleet: list[RobotConfig]) -> None:
    """Update the fleet configuration dynamically."""

def _get_robot_session(self, robot_id: str) -> RobotSession:
    """Get the InOrbit robot session for publishing/receiving."""
```

### Overridable Methods

```python
def _is_fleet_robot_online(self, robot_id: str) -> bool:
    """Override to check if a robot is reachable. Default: True."""

def _handle_command_exception(
    self, exception: Exception, command_name: str,
    robot_id: str, args: list, options: dict
) -> None:
    """Override to customize command error handling."""
```

### Lifecycle

```python
def start(self) -> None:
    """Start the connector (runs event loop in separate thread)."""

def stop(self) -> None:
    """Signal the connector to stop."""

def join(self) -> None:
    """Block until the connector finishes."""
```

## Connector (Single-Robot)

```python
from inorbit_connector.connector import Connector
```

Simpler variant for single-robot connectors. Extends `FleetConnector`.

### Constructor

```python
def __init__(self, robot_id: str, config: ConnectorRootConfig, **kwargs) -> None:
```

Adds `robot_id: str` parameter. Stores as `self.robot_id`.

### Abstract Methods

```python
@abstractmethod
async def _inorbit_command_handler(
    self, command_name: str, args: list, options: dict
) -> None:
    """Handle commands. No robot_id parameter (single robot)."""
```

Note: `_inorbit_robot_command_handler` is implemented internally to delegate to `_inorbit_command_handler`.

### Simplified Publishing (no robot_id)

```python
def publish_pose(
    self, x: float, y: float, yaw: float, frame_id: str, *args, **kwargs
) -> None:

def publish_odometry(self, **kwargs) -> None:

def publish_key_values(self, **kwargs) -> None:

def publish_system_stats(self, **kwargs) -> None:

def publish_map(self, frame_id: str, is_update: bool = False) -> None:
```

### Simplified Map Fetching

```python
async def fetch_map(self, frame_id: str) -> MapConfigTemp | None:
    """Override instead of fetch_robot_map for single-robot."""
```

### Session Access

```python
def _get_session(self) -> RobotSession:
    """Get the robot session (no robot_id needed)."""

def _is_robot_online(self) -> bool:
    """Override to check connectivity. Default: True."""
```

## Configuration Models

```python
from inorbit_connector.models import (
    ConnectorRootConfig,
    ConnectorSpecificConfig,
    RobotConfig,
    MapConfig,
    MapConfigTemp,
)
```

> **v3 change**: The old `ConnectorConfig(BaseModel)` is gone. Configuration is now built from
> two pydantic-settings `BaseSettings` classes — `ConnectorRootConfig` (top level) and
> `ConnectorSpecificConfig` (vendor config). They resolve `INORBIT_*` environment variables and
> `config/.env` **at instantiation time**, with init kwargs (YAML) taking precedence. There is no
> `account_id` field anymore (the account ID is resolved automatically by the edge SDK).

### ConnectorRootConfig

Generic over the vendor config type. Parametrize it: `ConnectorRootConfig[MyTargetConfig]`.

```python
class ConnectorRootConfig(BaseSettings, Generic[T]):
    # model_config: env_prefix="INORBIT_", env_file="config/.env",
    #               env_ignore_empty=True, case_sensitive=False, extra="ignore"

    api_key: str | None = None                       # INORBIT_API_KEY
    connection_config_url: HttpUrl = ...             # INORBIT_CONNECTION_CONFIG_URL (MQTT config endpoint)
    api_url: HttpUrl = ...                           # INORBIT_API_URL (REST API, default INORBIT_DEFAULT_API_URL)
    connector_type: str
    connector_config: T                              # a ConnectorSpecificConfig subclass
    use_websockets: bool = False
    update_freq: float = 1.0                         # Hz
    location_tz: str = "UTC"
    logging: LoggingConfig = LoggingConfig()         # logging.log_level (no top-level log_level)
    user_scripts_dir: DirectoryPath | None = None
    inorbit_robot_key: str | None = None
    maps: dict[str, MapConfig] = {}
    env_vars: dict[str, str] = {}
    metrics: MetricsConfig = MetricsConfig()
    fleet: list[RobotConfig]
```

Built-in behavior (do not re-implement in subclasses):
- Requires `api_key` **or** `inorbit_robot_key`, else raises `ValidationError` at instantiation.
- Validates `connector_type` equals the `CONNECTOR_TYPE` class var of the `connector_config` class.
- Enforces non-empty fleet and unique `robot_id` across the fleet.
- `to_singular_config(robot_id) -> Self` filters the config down to a single robot.
- Pass `_env_file=None` (or an explicit path) to control dotenv reading for root **and** nested
  config — used in tests to avoid reading `config/.env`.

### ConnectorSpecificConfig

Base for vendor-specific (fleet-wide) settings. Subclass it and set `CONNECTOR_TYPE`; the env-var
prefix is derived automatically as `INORBIT_{CONNECTOR_TYPE}_`. **Do not** hand-write a
`model_config = SettingsConfigDict(env_prefix=...)` — the base wires the prefix from `CONNECTOR_TYPE`.

```python
class ConnectorSpecificConfig(BaseSettings):
    CONNECTOR_TYPE: ClassVar[str]
    # env prefix auto-derived: INORBIT_{CONNECTOR_TYPE}_
    # model_config: env_ignore_empty=True, case_sensitive=False,
    #               env_file="config/.env", extra="ignore"
```

### RobotConfig

```python
class RobotConfig(BaseModel):
    robot_id: str
    cameras: list[CameraConfig] = []
```

Extend this for per-robot settings (e.g., `fleet_robot_id`, credentials).

### MapConfigTemp

```python
class MapConfigTemp(MapConfigBase):
    """Return from fetch_robot_map() with in-memory image."""
    image: bytes       # PNG image bytes
    # Inherited: map_id, origin_x, origin_y, resolution, map_label, format_version
```

### MapConfig

```python
class MapConfig(MapConfigBase):
    """For maps defined in YAML config with file paths."""
    file: FilePath     # Path to PNG file
```

## Command Handling

```python
from inorbit_connector.commands import (
    CommandFailure,
    CommandResultCode,
    parse_custom_command_args,
    CommandModel,
    ExcludeUnsetMixin,
)
```

### parse_custom_command_args

```python
def parse_custom_command_args(custom_command_args) -> tuple[str, dict[str, Any]]:
    """Parse custom command arguments.

    Returns:
        (script_name, argument_dict) tuple.

    Raises:
        ValueError: If arguments are not compliant.
        CommandFailure: If arguments cannot be parsed.
    """
```

### CommandFailure

```python
class CommandFailure(Exception):
    """Raise to report command failure to InOrbit."""

    def __init__(self, execution_status_details: str, stderr: str):
        self.execution_status_details = execution_status_details
        self.stderr = stderr
```

### CommandModel

```python
class CommandModel(BaseModel):
    """Base for command argument validation. Raises CommandFailure on ValidationError."""
    model_config = ConfigDict(extra="forbid")
```

### ExcludeUnsetMixin

```python
class ExcludeUnsetMixin:
    """Mixin that excludes unset fields from model_dump() by default."""
```

### CommandResultCode

```python
class CommandResultCode(str, Enum):
    SUCCESS = "0"
    FAILURE = "1"
```

## Utilities

```python
from inorbit_connector.utils import read_yaml
```

```python
def read_yaml(fname: str) -> dict:
    """Read a YAML configuration file. (v3: the `robot_id` parameter was removed.)"""
```

## Generated Project Structure

After running `pipx run cookiecutter gh:inorbit-ai/inorbit-connector-cookiecutter`:

```
<connector_slug>_connector/
├── <connector_slug>_connector/
│   ├── __init__.py
│   ├── <connector_slug>_connector.py    # Entry point with start()
│   └── src/
│       ├── __init__.py
│       ├── connector.py                 # Main connector (FleetConnector subclass, prefilled)
│       ├── commands.py                  # CustomScripts enum + CommandModel imports (prefilled)
│       ├── api/                         # API integration — ADD during implementation
│       │   ├── __init__.py
│       │   ├── client.py
│       │   └── data_poller.py
│       └── config/
│           ├── __init__.py
│           └── models.py                # Config models (ConnectorRootConfig / ConnectorSpecificConfig)
├── tests/
│   ├── __init__.py
│   ├── conftest.py                      # _clean_inorbit_env + _fast_asyncio_sleep fixtures
│   └── test_config_models.py
├── config/
│   ├── fleet.example.yaml
│   ├── example.env
│   └── user_scripts/                    # example user scripts
├── cac/                                 # CaC definitions (prefilled skeleton — fill in)
│   ├── actions.yaml
│   ├── data_sources.yaml
│   ├── footprint.yaml
│   ├── mission_tracking.yaml
│   └── status_definition.yaml
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.example.yaml
│   └── build.sh
├── pyproject.toml
├── README.md
├── REUSE.toml
└── LICENSES/MIT.txt
```

> The cookiecutter generates `src/config/models.py`, `src/connector.py`, and `src/commands.py`
> already prefilled with the v3 patterns. The `src/api/` package is **not** generated — add it
> during implementation.
