# InOrbit Configuration-as-Code (CaC)

CaC defines how robots appear and behave in InOrbit. Files go in the `cac/` directory with multi-document YAML (separated by `---`). All objects use `apiVersion: v0.1`.

For full reference, see: https://developer.inorbit.ai/docs#configuring-data-sources

## Scope

Every CaC object has a `metadata.scope` that determines which robots it applies to:
- **Tag scope**: `tag/[account_id]/[tag_id]` — applies to robots with that tag (most common)
- **Account scope**: `account/[account_id]` — applies to all robots in the account (used for MissionDefinition and edge execution actions)

Replace `[account_id]` and `[tag_id]` with actual values.

## ActionDefinition

Defines commands available in the InOrbit UI. Each action maps to a custom command script name handled by the connector.

### Simple Command (no parameters)

```yaml
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: pause-robot
  scope: tag/[account_id]/[tag_id]
spec:
  type: RunScript
  label: Pause Robot
  group: Robot Control
  arguments:
    - name: filename
      type: string
      value: pause_robot           # Must match CustomScripts enum value
  confirmation:
    required: false
  description: Pause the robot
  lock: false
```

### Command with Parameters

```yaml
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: queue-fleet-mission
  scope: tag/[account_id]/[tag_id]
spec:
  type: RunScript
  label: Queue Fleet Mission
  group: Missions
  arguments:
    - name: filename
      type: string
      value: queue_fleet_mission
    - name: mission_id
      type: string
      value: ""
      input:
        control: text              # Free text input
    - name: assign_to_robot
      type: string
      value: "False"
      input:
        control: select            # Dropdown
        values:
          - label: "True"
            value: "True"
          - label: "False"
            value: "False"
  confirmation:
    required: false
  description: Queue a mission on the fleet manager
  lock: false
```

### Command with Confirmation

```yaml
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: abort-all-missions
  scope: tag/[account_id]/[tag_id]
spec:
  type: RunScript
  label: Abort All Missions
  group: Missions
  arguments:
    - name: filename
      type: string
      value: abort_all_missions
  confirmation:
    required: true                 # User must confirm before execution
  description: Abort all pending missions
  lock: false
```

### Edge Mission Execution Actions (account scope)

**Only include these if the connector uses edge mission execution.** All three are required together.

```yaml
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: executeMission
  scope: account/[account_id]
spec:
  type: RunScript
  label: Execute edge mission
  group: Reserved
  lock: false
  confirmation:
    required: false
  description: Send mission to Edge robot
  arguments:
    - name: filename
      type: string
      value: executeMissionAction
    - input:
        control: text
      name: robotId
      type: string
    - input:
        control: text
      name: missionId
      type: string
    - input:
        control: text
      name: missionArgs
      type: string
    - input:
        control: text
      name: options
      type: string
    - input:
        control: text
      name: missionDefinition
      type: string
---
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: cancelMission
  scope: account/[account_id]
spec:
  type: RunScript
  label: Cancel edge mission
  group: Reserved
  lock: false
  confirmation:
    required: false
  description: Cancel mission on Edge robot
  arguments:
    - name: filename
      type: string
      value: cancelMissionAction
    - input:
        control: text
      name: missionId
      type: string
---
apiVersion: v0.1
kind: ActionDefinition
metadata:
  id: updateMission
  scope: account/[account_id]
spec:
  type: RunScript
  label: Update edge mission
  group: Reserved
  lock: false
  confirmation:
    required: false
  description: Update mission on Edge robot
  arguments:
    - name: filename
      type: string
      value: updateMissionAction
    - input:
        control: text
      name: missionId
      type: string
    - input:
        control: text
      name: action
      type: string
```

### Action Spec Fields

| Field | Description |
|-------|-------------|
| `type` | `RunScript` (most common) or `PublishToTopic` |
| `label` | Display name in UI |
| `group` | Groups actions in the UI menu |
| `description` | Tooltip/help text |
| `confirmation.required` | `true` to require user confirmation |
| `lock` | `true` to prevent simultaneous execution |
| `arguments[].name` | `filename` is the script name; others are parameters |
| `arguments[].input.control` | `text`, `select`, or `checkbox` |

## DataSourceDefinition

Defines data sources for InOrbit dashboards, timelines, and status panels.

### Key-Value Source

```yaml
apiVersion: v0.1
kind: DataSourceDefinition
metadata:
  id: battery_percent
  scope: tag/[account_id]/[tag_id]
spec:
  label: Battery
  source:
    keyValue:
      key: battery_percent         # Must match key published via publish_robot_key_values()
  timeline:
    fieldType: number
  unit: "%"
```

### String Key-Value (no timeline)

```yaml
apiVersion: v0.1
kind: DataSourceDefinition
metadata:
  id: state_text
  scope: tag/[account_id]/[tag_id]
spec:
  label: State
  source:
    keyValue:
      key: state_text
  timeline: {}
```

### Boolean Key-Value

```yaml
apiVersion: v0.1
kind: DataSourceDefinition
metadata:
  id: api_connected
  scope: tag/[account_id]/[tag_id]
spec:
  label: API Connected
  source:
    keyValue:
      key: api_connected
  timeline:
    fieldType: boolean
```

### Derived (Computed) Source

Uses InOrbit's safe expression language to compute values from other data sources:

```yaml
apiVersion: v0.1
kind: DataSourceDefinition
metadata:
  id: mission_status
  scope: tag/[account_id]/[tag_id]
spec:
  label: Mission Status
  source:
    derived:
      language: safe
      transform: |
        api_connected = getValue("api_connected");
        api_connected == false ? "Error" :
        (
          mission_text = getValue("mission_text");
          state_text = getValue("state_text");
          state_text == "Executing"
            ? (match("Charging", mission_text) ? "Charging" : "Mission")
            : ((state_text == "EmergencyStop" or state_text == "Error")
                ? "Error"
                : "Idle")
        )
  timeline: {}
```

### Mission Tracking Data Source

Required for connectors that publish `mission_tracking` key-value:

```yaml
apiVersion: v0.1
kind: DataSourceDefinition
metadata:
  id: mission_tracking
  scope: tag/[account_id]/[tag_id]
spec:
  label: Mission Tracking
  source:
    keyValue:
      key: mission_tracking
  timeline: {}
```

## MissionTracking

Configures how InOrbit tracks missions for this connector.

### Without Edge Execution (cloud API only)

For connectors that track missions reported by the target system but don't execute missions locally:

```yaml
apiVersion: v0.1
kind: MissionTracking
metadata:
  id: all
  scope: tag/[account_id]/[tag_id]
spec:
  processingType:
    - api
  attributes:
    mission_tracking:
      - type: missionApi
  stateDefinitions:
    Executing:
      defaultStatus: OK
```

### With Edge Execution

For connectors that execute missions locally via the edge executor:

```yaml
apiVersion: v0.1
kind: MissionTracking
metadata:
  id: all
  scope: tag/[account_id]/[tag_id]
spec:
  processingType:
    - api
  autoClosePreviousMission: true
  execution:
    executor:
      type: edge
      options:
        coordinateSystem: robot    # or "world"
  # Optional: also track missions reported by the target system
  # attributes:
  #   mission_tracking:
  #     - type: missionApi
```

**Key fields:**
- `processingType`: Always `["api"]`
- `autoClosePreviousMission`: `true` to auto-close previous mission when a new one starts
- `execution.executor.type`: `edge` for local execution
- `execution.executor.options.coordinateSystem`: `robot` (coordinates relative to robot) or `world`
- `attributes`: Links a data source to mission tracking (for missions reported by the target system)
- `stateDefinitions`: Custom status mapping for mission states

## MissionDefinition

**Only for connectors with edge mission execution.** Defines available mission templates in InOrbit.

### Single-Step Mission

```yaml
apiVersion: v0.1
kind: MissionDefinition
metadata:
  id: navigate_to_goal
  scope: account/[account_id]
spec:
  label: "Navigate to Goal"
  steps:
    - label: "Navigate to goal"
      timeoutSecs: 60
      runAction:
        actionId: gotoGoals        # Must match an ActionDefinition id
        arguments:
          goals: "Goal1"
```

### Multi-Step Mission

```yaml
apiVersion: v0.1
kind: MissionDefinition
metadata:
  id: patrol_route
  scope: account/[account_id]
spec:
  label: "Patrol Route"
  steps:
    - label: "Go to Point A"
      timeoutSecs: 60
      runAction:
        actionId: gotoGoals
        arguments:
          goals: "PointA"
    - label: "Wait 5 seconds"
      timeoutSecs: 5
    - label: "Go to Point B"
      timeoutSecs: 60
      runAction:
        actionId: gotoGoals
        arguments:
          goals: "PointB"
```

### Mission with Data Step

```yaml
apiVersion: v0.1
kind: MissionDefinition
metadata:
  id: high_priority_nav
  scope: account/[account_id]
spec:
  label: "High Priority Navigate"
  steps:
    - label: "Set Priority"
      data:
        priority: 1
    - label: "Navigate"
      timeoutSecs: 60
      runAction:
        actionId: gotoGoals
        arguments:
          goals: "Goal1"
```

### Step Types

| Step Type | Fields | Description |
|-----------|--------|-------------|
| `runAction` | `actionId`, `arguments` | Execute an action on the robot |
| `data` | key-value pairs | Set metadata on the mission |
| (wait) | `timeoutSecs` only (no action) | Wait for a duration |

## RobotFootprint

Defines the robot's physical dimensions for visualization and collision avoidance.

```yaml
apiVersion: v0.1
kind: RobotFootprint
metadata:
  id: all
  scope: tag/[account_id]/[tag_id]
spec:
  footprint:                       # Robot body polygon (meters)
    - x: -0.295
      y: 0.29
    - x: -0.295
      y: -0.29
    - x: 0.595
      y: -0.29
    - x: 0.595
      y: 0.29
  bufferFootprint:                 # Safety buffer polygon (larger)
    - x: -0.795
      y: 0.79
    - x: -0.795
      y: -0.79
    - x: 1.095
      y: -0.79
    - x: 1.095
      y: 0.79
  radius: 0.3                     # Approximate radius for simple checks
```

## Preferences

Configures UI panels and features for the robot type.

```yaml
apiVersion: v0.1
kind: Preferences
metadata:
  id: all
  scope: tag/[account_id]/[tag_id]
spec:
  panels:
    teleop: false                  # Disable teleop if not supported
```

## What to Include

Not all CaC types apply to every connector. Use this guide:

| CaC Object | When to Include |
|-------------|----------------|
| ActionDefinition | Always — define all custom commands |
| DataSourceDefinition | Always — define all published key-values |
| MissionTracking | When the connector tracks or executes missions |
| MissionDefinition | Only with edge mission execution |
| RobotFootprint | When robot dimensions are known |
| Preferences | When default UI panels need adjustment |
| executeMission/cancelMission/updateMission | **Only** with edge mission execution |
