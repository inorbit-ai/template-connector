# Edge Mission Execution

**This file is only relevant when the connector needs edge mission execution** — i.e., missions dispatched from InOrbit and executed locally via behavior trees. If missions are only tracked via cloud API, this file is not needed.

Based on `inorbit_edge_executor` v4.0.1. For full source, see `github.com/inorbit-ai/inorbit_edge_executor`.

## When to Use

Use edge mission execution when:
- The target system has mission/task/job concepts
- Missions should be dispatched from InOrbit and executed on the edge (not just tracked)
- You need behavior trees for multi-step mission orchestration

## Architecture

```
InOrbit Cloud → executeMission command → Connector command handler
    → WorkerPool.submit_work(mission, options, shared_memory)
        → Worker → BehaviorTree (built from mission steps)
            → RunActionNode / WaitNode / SetDataNode / etc.
```

**Pipeline**: `WorkerPool` manages `Worker` instances, each executing a `BehaviorTree` built from mission step definitions.

## Setup Pattern

```python
from inorbit_edge_executor.worker_pool import WorkerPool
from inorbit_edge_executor.inorbit import InOrbitAPI
from inorbit_edge_executor.db import get_db


class MyFleetConnector(FleetConnector):

    @override
    async def _connect(self) -> None:
        # ... set up API client, data poller ...

        # Edge executor setup
        api = InOrbitAPI(api_key=self._config.api_key)
        db = await get_db("sql:missions.db")  # or "dummy" for dev
        await db.connect()

        self._worker_pool = WorkerPool(api=api, db=db)
        await self._worker_pool.start()

    @override
    async def _disconnect(self) -> None:
        if self._worker_pool:
            await self._worker_pool.shutdown()
        # ... clean up other resources ...
```

## Submitting Missions

When the connector receives an `executeMissionAction` command:

```python
from inorbit_edge_executor.datatypes import MissionRuntimeOptions, MissionRuntimeSharedMemory
from inorbit_edge_executor.mission import Mission

async def _handle_execute_mission(self, robot_id: str, args: dict) -> dict:
    mission = Mission.model_validate_json(args["missionDefinition"])
    options = MissionRuntimeOptions.model_validate_json(args.get("options", "{}"))
    shared_memory = MissionRuntimeSharedMemory()

    result = await self._worker_pool.submit_work(
        mission=mission,
        options=options,
        shared_memory=shared_memory,
    )
    return result  # {"id": mission_id} on success, {"error": msg} on failure
```

## Mission Control

```python
# Abort a running mission
result = self._worker_pool.abort_mission(mission_id)
# Returns {"id": mission_id, "cancelled": True} if cancelled

# Pause a running mission
await self._worker_pool.pause_mission(mission_id)

# Resume a paused mission
await self._worker_pool.resume_mission(mission_id)
```

## MissionState Enum

```python
from inorbit_edge_executor.inorbit import MissionState

class MissionState(Enum):
    starting = "starting"
    in_progress = "in-progress"
    paused = "paused"
    completed = "completed"
    aborted = "aborted"
    abandoned = "abandoned"
```

## Step Types

Mission definitions contain steps, each of a specific type:

| Step Type | Class | Description |
|-----------|-------|-------------|
| `runAction` | `MissionStepRunAction` | Execute an action on the robot |
| `data` / `setData` | `MissionStepSetData` | Set metadata on the mission |
| `wait` | `MissionStepWait` | Wait for a duration (via `timeoutSecs`) |
| `waitUntil` | `MissionStepWaitUntil` | Wait until an expression is true |
| `poseWaypoint` | `MissionStepPoseWaypoint` | Navigate to a pose (x, y, theta) |
| `if` | `MissionStepIf` | Conditional execution based on expression |

```python
from inorbit_edge_executor.datatypes import (
    MissionStepRunAction,
    MissionStepSetData,
    MissionStepWait,
    MissionStepWaitUntil,
    MissionStepPoseWaypoint,
    MissionStepIf,
    Pose,
    Target,
)
```

## Custom Node Builders

Override `NodeFromStepBuilder` to customize how mission steps are converted to behavior tree nodes:

```python
from inorbit_edge_executor.behavior_tree import (
    BehaviorTreeBuilderContext,
    NodeFromStepBuilder,
    DefaultTreeBuilder,
    RunActionNode,
)


class MyNodeBuilder(NodeFromStepBuilder):
    """Custom step-to-node conversion for MyTarget."""

    def __init__(self, context: BehaviorTreeBuilderContext) -> None:
        super().__init__(context)

    def visit_run_action(self, step) -> RunActionNode:
        """Customize how runAction steps are converted."""
        # Example: translate action arguments for target system
        return super().visit_run_action(step)


# Wire into WorkerPool via custom TreeBuilder
class MyTreeBuilder(DefaultTreeBuilder):
    def __init__(self) -> None:
        super().__init__(step_builder_factory=MyNodeBuilder)
```

Then pass to WorkerPool:

```python
self._worker_pool = WorkerPool(
    api=api,
    db=db,
    behavior_tree_builder=MyTreeBuilder(),
)
```

## Custom WorkerPool

Override `WorkerPool` for custom mission translation or context:

```python
from inorbit_edge_executor.worker_pool import WorkerPool
from inorbit_edge_executor.behavior_tree import BehaviorTreeBuilderContext
from inorbit_edge_executor.mission import Mission


class MyWorkerPool(WorkerPool):

    def translate_mission(self, mission: Mission) -> Mission:
        """Transform mission before execution (e.g., resolve waypoint names)."""
        return mission

    def create_builder_context(self) -> BehaviorTreeBuilderContext:
        """Provide custom context to behavior tree builders."""
        return BehaviorTreeBuilderContext()
```

## Persistence

- **Production**: `SqliteDB` — persists worker state across restarts
  ```python
  db = await get_db("sql:missions.db")
  ```
- **Development**: `DummyDB` — in-memory, no persistence
  ```python
  db = await get_db("dummy")
  ```

Both implement `WorkerPersistenceDB`:
```python
await db.connect()
# ... use with WorkerPool ...
await db.shutdown()
```

## Wiring into Connector

### Command Handler Integration

```python
from inorbit_connector.commands import parse_custom_command_args, CommandFailure

from .commands import CustomScripts


async def _inorbit_robot_command_handler(
    self, robot_id: str, command_name: str, args: list, options: dict
) -> None:
    result_function = options["result_function"]

    if command_name == "customCommand":
        script_name, script_args = parse_custom_command_args(args)

        match script_name:
            case CustomScripts.EXECUTE_MISSION_ACTION:
                result = await self._handle_execute_mission(robot_id, script_args)
                if "error" in result:
                    raise CommandFailure(
                        execution_status_details=result["error"],
                        stderr=result["error"],
                    )
            case CustomScripts.CANCEL_MISSION_ACTION:
                self._worker_pool.abort_mission(script_args["missionId"])
            case CustomScripts.UPDATE_MISSION_ACTION:
                action = script_args["action"]
                mission_id = script_args["missionId"]
                if action == "pause":
                    await self._worker_pool.pause_mission(mission_id)
                elif action == "resume":
                    await self._worker_pool.resume_mission(mission_id)
            # ... other custom commands ...

        result_function(CommandResultCode.SUCCESS)
```

### Command Enum

```python
from enum import StrEnum

class CustomScripts(StrEnum):
    # ... other commands ...
    EXECUTE_MISSION_ACTION = "executeMissionAction"
    CANCEL_MISSION_ACTION = "cancelMissionAction"
    UPDATE_MISSION_ACTION = "updateMissionAction"
```

## Required CaC

When using edge mission execution, the following CaC objects are **required** (see `reference/cac-objects.md`):

1. **executeMission**, **cancelMission**, **updateMission** ActionDefinitions (account scope)
2. **MissionTracking** with `execution.executor.type: edge`
3. **MissionDefinition** objects for each available mission template

## Key Imports

```python
# Worker Pool
from inorbit_edge_executor.worker_pool import WorkerPool

# Database
from inorbit_edge_executor.db import get_db
from inorbit_edge_executor.sqlite_backend import SqliteDB
from inorbit_edge_executor.dummy_backend import DummyDB

# InOrbit API
from inorbit_edge_executor.inorbit import InOrbitAPI, MissionState, MissionStatus

# Mission
from inorbit_edge_executor.mission import Mission
from inorbit_edge_executor.datatypes import (
    MissionRuntimeOptions,
    MissionRuntimeSharedMemory,
    MissionStepTypes,
    MissionStepRunAction,
    MissionStepSetData,
    MissionStepWait,
    MissionStepWaitUntil,
    MissionStepPoseWaypoint,
    MissionStepIf,
    Pose,
    Target,
)

# Behavior Trees
from inorbit_edge_executor.behavior_tree import (
    BehaviorTree,
    BehaviorTreeSequential,
    BehaviorTreeErrorHandler,
    BehaviorTreeBuilderContext,
    TreeBuilder,
    DefaultTreeBuilder,
    NodeFromStepBuilder,
    RunActionNode,
    WaitNode,
    SetDataNode,
    TimeoutNode,
)

# Exceptions
from inorbit_edge_executor.exceptions import (
    RobotBusyException,
    MissionNotFoundException,
    InvalidMissionStateException,
)
```
