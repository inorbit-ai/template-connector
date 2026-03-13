# Framework API Reference

Based on `inorbit-connector-python` v2.2.0. For full source, see `github.com/inorbit-ai/inorbit-connector-python`.

## FleetConnector

```python
from inorbit_connector.connector import FleetConnector
```

Base class for connectors managing multiple robots through a fleet API.

### Constructor

```python
def __init__(self, config: ConnectorConfig, **kwargs) -> None:
```

**Keyword arguments:**
- `register_user_scripts` (bool): Default `False`
- `default_user_scripts_dir` (str): Default `"~/.inorbit_connectors/connector-{ClassName}/local/"`
- `create_user_scripts_dir` (bool): Default `False`
- `register_custom_command_handler` (bool): Default `True`
- `publish_connector_system_stats` (bool): Default `False`

### Properties

- `robot_ids -> list[str]`: Robot IDs from configuration
- `_config`: The `ConnectorConfig` object
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
def __init__(self, robot_id: str, config: ConnectorConfig, **kwargs) -> None:
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
from inorbit_connector.models import ConnectorConfig, RobotConfig, MapConfig, MapConfigTemp
```

### ConnectorConfig

```python
class ConnectorConfig(BaseModel):
    api_key: str | None = os.getenv("INORBIT_API_KEY")
    api_url: HttpUrl = os.getenv("INORBIT_API_URL", INORBIT_CLOUD_SDK_ROBOT_CONFIG_URL)
    connector_type: str
    connector_config: BaseModel          # Your target-specific config
    update_freq: float = 1.0             # Hz
    location_tz: str = "UTC"
    logging: LoggingConfig = LoggingConfig()
    user_scripts_dir: DirectoryPath | None = None
    account_id: str | None = None
    inorbit_robot_key: str | None = None
    maps: dict[str, MapConfig] = {}
    env_vars: dict[str, str] = {}
    fleet: list[RobotConfig]
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
def read_yaml(fname: str, robot_id: str = None) -> dict:
    """Read a YAML configuration file."""
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
│       ├── connector.py                 # Main connector (add this)
│       ├── commands.py                  # Command enums/models (add this)
│       ├── api/                         # API integration (add this)
│       │   ├── __init__.py
│       │   ├── client.py
│       │   └── data_poller.py
│       └── config/
│           ├── __init__.py
│           └── models.py                # Pydantic config models
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_config_models.py
├── config/
│   ├── fleet.example.yaml
│   └── example.env
├── cac/                                 # CaC definitions (add this)
│   └── *.yaml
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.example.yaml
│   └── build.sh
├── pyproject.toml
├── README.md
└── REUSE.toml
```
