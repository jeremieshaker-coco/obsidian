# Device API Redesign v3

> **Summary**: This design supersedes v2 by introducing a cleaner separation of concerns:
> 1. **Three orthogonal dimensions**: Deployment status, trip status, and operational status (grounded)
> 2. **Issues as the grounding mechanism**: `grounded` is derived from active device issues, not a commanded state
> 3. **Device Issues module in Operations Service**: Not a separate service, but a new module with shared DB access
> 4. **Semantic actions for UIs**: Replace low-level state transitions with business-meaningful operations
> 5. **Simplified event-sourced state**: `RobotStateHistory` becomes audit-only; operational state is derived from source-of-truth tables

---

## Problem Statement

Our current device state management has several architectural issues:

- **Fragmented State Ownership**: Device state is scattered across multiple services with no single source of truth
- **Tight Coupling to External Systems**: Multiple services directly consume raw MQTT events
- **Blurred Service Boundaries**: Services have overlapping responsibilities
- **Conflated State Dimensions**: `operation_state` tries to encode deployment, maintenance and trip status in a single enum
- **Event-sourced flags that should be derived**: Fields like `needsMaintenance`, `needsMovement`, and `driveable` are stored when they could be computed
- **Unused fields**: `driveable` is never set in production code

### The Core Problem with `operation_state`

The current `operation_state` enum (`OFF_DUTY`, `PARKED`, `ON_TRIP`, `GROUNDED`) conflates three orthogonal dimensions:

| Dimension              | Values               | Meaning                              |
| ---------------------- | -------------------- | ------------------------------------ |
| **Deployment Status**  | UNDEPLOYED, DEPLOYED | Where is the robot in its lifecycle? |
| **Operational Status** | OK, GROUNDED         | Can the robot operate?               |
| **Trip Status**        | IDLE, ON_TRIP        | Is the robot currently on a trip?    |

This causes problems:
- When a PARKED robot is grounded, we lose track that it was deployed
- When ungrounded, operators must manually choose PARKED vs OFF_DUTY
- The state machine enforces transitions that don't match business intent

---

## Design Principles

1. **Single Responsibility**: Each service owns one domain
2. **Facts vs Interpretation**: Device Service reports physical facts; other services interpret them
3. **Derived State**: Operational flags are computed from source-of-truth tables, not stored directly
4. **Orthogonal Dimensions**: Deployment and operational status are tracked separately
5. **Semantic Actions**: UIs call business-meaningful endpoints, not state transitions
6. **Backward Compatibility**: Continue writing `operation_state` to shadow for robot firmware
7. **Audit vs Derivation**: History tables are for audit trails, not for deriving current state

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ROBOT SOFTWARE                                 │
│  Reads: operation_state (legacy), light_mode, ota_enabled (new)             │
│  Publishes: hasCargo (new - replaces hasFood inference)                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │ AWS IoT Shadow
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DEVICE SERVICE (NEW)                                 │
│                                                                             │
│  Owns: Physical state (telemetry, connectivity, component health)           │
│  Does: Writes shadow fields when instructed by Operations Service           │
│  Publishes: Device.Heartbeat, Device.HealthDegraded, Device.HealthRestored  │
│  Note: Absorbs telemetry/connectivity from current State Service            │
└─────────────────────────────────────────────────────────────────────────────┘
        │                                               
        │ Device.HealthDegraded, Device.Heartbeat                        
        ▼                                               
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OPERATIONS SERVICE                                  │
│                                                                             │
│  ┌─────────────────────────────┐    ┌────────────────────────────────────┐  │
│  │   Device Issues Module      │    │   Trips / Deployments / FO Tasks   │  │
│  │   (NEW)                     │    │   (Existing)                       │  │
│  │                             │    │                                    │  │
│  │   - Issue lifecycle         │    │   - Trip lifecycle                 │  │
│  │   - Operator-raised issues  │    │   - Deployment lifecycle           │  │
│  │   - FO request association  │    │   - FO task management             │  │
│  └──────────────┬──────────────┘    └─────────────────┬──────────────────┘  │
│                 │                                     │                     │
│                 └──────────────┬──────────────────────┘                     │
│                                ▼                                            │
│                   ┌────────────────────────────┐                            │
│                   │  Operational State Deriver │                            │
│                   │                            │                            │
│                   │  Computes (all derived):   │                            │
│                   │  - deployment_status       │                            │
│                   │  - trip_status             │                            │
│                   │  - grounded                │                            │
│                   │  - needs_maintenance       │                            │
│                   │  - operation_state (legacy)│                            │
│                   │                            │                            │
│                   │  Calls: DeviceService      │                            │
│                   │         to write shadow    │                            │
│                   └────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LID SERVICE (née State Service)                          │
│                                                                             │
│  Owns: Lid cycles, PIN entry flows                                          │
│  Publishes: LidCycle.*, Robots.PinEntry                                     │
│  Note: Operation state logic removed; becomes focused lid management        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Service Boundary Changes

| Current Service | New Service | What Moves |
|-----------------|-------------|------------|
| State Service | **Device Service** (new) | Telemetry, connectivity, component health, shadow writes |
| State Service | **Lid Service** (renamed) | Lid cycles, PIN entry |
| State Service | **Operations Service** | Operation state logic, state machine (becomes issue-based) |

### Why Device Issues Module Lives in Operations Service

1. **Shared context**: Issues often trigger FO tasks (which Operations already owns)
2. **Related patterns**: Operations already has `FoAssistanceRequest`, `FoTask` 
3. **Simpler infrastructure**: No new service to deploy/monitor
4. **Transactional operations**: Can create issue + FO task in single transaction

---

## State Model

### The Three Dimensions

```typescript
interface RobotOperationalState {
  // Dimension 1: Deployment lifecycle
  deployment_status: 'UNDEPLOYED' | 'DEPLOYED';
  
  // Dimension 2: Trip lifecycle  
  trip_status: 'IDLE' | 'ON_TRIP';
  
  // Dimension 3: Operational status (derived from issues)
  grounded: boolean;
  
  // Derived for backward compatibility with robot firmware
  operation_state: 'OFF_DUTY' | 'PARKED' | 'ON_TRIP' | 'GROUNDED';
}
```

### Derivation Logic

All operational flags are **computed from source-of-truth tables**, not stored:

```typescript
// Operations Service - Operational State Deriver
async function deriveOperationalState(serial: string): Promise<RobotOperationalState> {
  // Query source-of-truth tables
  const [deployment, trip, issues, foTasks] = await Promise.all([
    deploymentRepo.getActiveDeployment(serial),
    tripRepo.getActiveTrip(serial),
    deviceIssuesRepo.listActiveIssues(serial),
    foTasksRepo.getActiveTasks(serial),
  ]);
  
  // Compute dimensions from facts
  const deploymentStatus = deployment ? 'DEPLOYED' : 'UNDEPLOYED';
  const tripStatus = trip ? 'ON_TRIP' : 'IDLE';
  const grounded = issues.some(i => i.grounds_robot);
  
  // Derive additional flags (replaces stored needsMaintenance, etc.)
  const needsMaintenance = issues.some(i => i.severity === 'WARNING') || 
                           foTasks.some(t => t.state === 'PENDING');
  const undergoingMaintenance = foTasks.some(t => t.state === 'IN_PROGRESS');
  
  // Legacy operation_state derivation
  let operationState: string;
  if (grounded) {
    operationState = 'GROUNDED';
  } else if (tripStatus === 'ON_TRIP') {
    operationState = 'ON_TRIP';
  } else if (deploymentStatus === 'DEPLOYED') {
    operationState = 'PARKED';
  } else {
    operationState = 'OFF_DUTY';
  }
  
  return {
    deployment_status: deploymentStatus,
    trip_status: tripStatus,
    grounded,
    needs_maintenance: needsMaintenance,
    undergoing_maintenance: undergoingMaintenance,
    operation_state: operationState,
  };
}
```

### Dimension Relationships

```
┌────────────────────────────────────────────────────────────────┐
│                     VALID STATE COMBINATIONS                   │
├─────────────────┬─────────────┬──────────┬─────────────────────┤
│ deployment      │ trip        │ grounded │ operation_state     │
│                 │             │          │ (legacy)            │
├─────────────────┼─────────────┼──────────┼─────────────────────┤
│ UNDEPLOYED      │ IDLE        │ false    │ OFF_DUTY            │
│ UNDEPLOYED      │ IDLE        │ true     │ GROUNDED            │
│ DEPLOYED        │ IDLE        │ false    │ PARKED              │
│ DEPLOYED        │ IDLE        │ true     │ GROUNDED            │
│ DEPLOYED        │ ON_TRIP     │ false    │ ON_TRIP             │
│ DEPLOYED        │ ON_TRIP     │ true     │ GROUNDED            │
├─────────────────┴─────────────┴──────────┴─────────────────────┤
│ Note: UNDEPLOYED + ON_TRIP is invalid (trip implies deployed)  │
└────────────────────────────────────────────────────────────────┘
```

### Key Behavior: Grounding Preserves Deployment

When a robot is grounded:
- `deployment_status` remains unchanged (still DEPLOYED if it was deployed)
- `grounded` becomes `true`
- `operation_state` becomes `GROUNDED` (for backward compat)

When issues are cleared:
- `grounded` becomes `false`
- `operation_state` returns to the deployment-based value (PARKED, OFF_DUTY, etc.)
- **No manual state selection required**

```
Timeline:
─────────────────────────────────────────────────────────────────────────────
Robot deployed to parking lot
  deployment_status: DEPLOYED
  trip_status: IDLE
  grounded: false
  operation_state: PARKED
                         │
                         ▼ GPS fault detected, issue raised
  deployment_status: DEPLOYED  (unchanged!)
  trip_status: IDLE            (unchanged!)
  grounded: true
  operation_state: GROUNDED
                         │
                         ▼ Issue cleared (GPS recovered or operator cleared)
  deployment_status: DEPLOYED  (still deployed!)
  grounded: false
  operation_state: PARKED      (automatically restored!)
─────────────────────────────────────────────────────────────────────────────
```

---

## Service Responsibilities

### Device Service (NEW)

This is a **new service** that absorbs telemetry/connectivity responsibilities from the current State Service.

**What it owns:**
- Device shadow (physical state)
- Device registry (serial numbers, hardware metadata)
- Connectivity status (online/offline, DriveU status)
- Component health snapshots (physical facts, not interpretations)

**What it does:**
- Consumes raw MQTT heartbeats from SQS
- Consumes DriveU webhooks
- Normalizes and persists device state
- Publishes clean domain events
- **Writes shadow fields when instructed** (by Operations Service)

**What it does NOT do:**
- Create or manage issues (that's Operations Service's Device Issues module)
- Make business decisions about grounding
- Derive operational state

**Events Published:**

| Event                      | Trigger                            | Purpose                                                      | Consumers |
| -------------------------- | ---------------------------------- | ------------------------------------------------------------ | --------- |
| `Device.Heartbeat`         | Every processed heartbeat (~5-10s) | Full state snapshot for cache builders                       | **Dispatch Engine**: Update robot supply cache (location, battery, service codes) for assignment decisions<br/>**Fleet Service**: Update Redis/DynamoDB cache for Uber position updates<br/>**Trip Monitor**: Update trip trace, recalculate ETA<br/>**Operations Service**: Update Redis ephemeral state |
| `Device.Moved`             | Location changed > 10m             | Route tracking, geofence detection                           | **Trip Monitor**: Route deviation detection, arrival detection<br/>**Operations Service**: Geofence checks (entering/leaving service areas) |
| `Device.LidOpened`         | Lid state: closed → open           | Lid cycle tracking, security alerts                          | **Lid Service**: Start lid cycle, track open duration<br/>**Deliveries Service**: Track loading start time |
| `Device.LidClosed`         | Lid state: open → closed           | Lid cycle tracking                                           | **Lid Service**: Complete lid cycle, emit `LidCycle.Completed`<br/>**Deliveries Service**: Trigger load completion if conditions met |
| `Device.LidJammed`         | Lid mechanism reports fault        | Critical fault handling                                      | **Lid Service**: Abort lid cycle, alert operations<br/>**Operations Service**: Create urgent FO task<br/>**Deliveries Service**: Handle failed handoff scenario |
| `Device.BatteryLow`        | Battery crosses 20% threshold      | Ops alerts, preemptive recharging                            | **Operations Service**: Create FO task for battery swap<br/>**Dispatch Engine**: Reduce robot availability score |
| `Device.BatteryCritical`   | Battery crosses 5% threshold       | Critical ops alerts, trip safety                             | **Operations Service**: Ground robot, create urgent FO task<br/>**Dispatch Engine**: Remove from available supply<br/>**Trip Monitor**: Alert if robot is mid-trip |
| `Device.ConnectivityChanged` | Online ↔ Offline transition      | Connectivity tracking                                        | **Operations Service**: Update robot connectivity status, potentially ground if offline too long<br/>**Dispatch Engine**: Remove/add to supply |
| `Device.HealthDegraded`    | Component enters fault state       | Trigger for Device Issues module to potentially create issue | **Operations Service** (Device Issues module): Create issue based on component severity<br/>**Dispatch Engine**: Mark robot unhealthy, remove from supply |
| `Device.HealthRestored`    | Component returns to OK            | Trigger for Device Issues module to potentially clear issue  | **Operations Service** (Device Issues module): Auto-clear issue if applicable<br/>**Dispatch Engine**: Mark robot healthy, add to supply |
| `Device.EmergencyStop`     | Emergency stop button pressed      | Safety-critical response                                     | **Operations Service**: Immediately ground robot, create critical alert<br/>**Dispatch Engine**: Remove from supply immediately<br/>**Trip Monitor**: Pause trip, alert if mid-delivery |
| `Device.PinEntry`          | Key pressed on device pin pad      | Lid unlock validation                                        | **Lid Service**: Validate PIN against active unlock context, decide whether to open lid |
| `Device.CargoChanged`      | `hasCargo` state changes           | Replaces `hasFood` inference from lid events                 | **Operations Service**: Update cargo status for trip tracking<br/>**Deliveries Service**: Update delivery state<br/>**Dispatch Engine**: Update supply cache |

**Shadow Write API:**

```protobuf
service DeviceService {
  // Shadow field updates (called by Operations Service)
  rpc SetShadowFields(SetShadowFieldsRequest) returns (SetShadowFieldsResponse);
  
  // Individual field RPCs (for granular updates)
  rpc UpdateLightMode(UpdateLightModeRequest) returns (UpdateLightModeResponse);
  rpc UpdateSoundProfile(UpdateSoundProfileRequest) returns (UpdateSoundProfileResponse);
  rpc UpdateOtaEnabled(UpdateOtaEnabledRequest) returns (UpdateOtaEnabledResponse);
  
  // Legacy: Will be removed after robot firmware migration
  rpc SetOperationState(SetOperationStateRequest) returns (SetOperationStateResponse);
}

message SetShadowFieldsRequest {
  string device_serial = 1;
  
  // New explicit fields
  optional LightMode light_mode = 2;
  optional SoundProfile sound_profile = 3;
  optional bool ota_enabled = 4;
  optional bool autonomy_enabled = 5;
  
  // Legacy field (deprecated, for backward compat only)
  optional string operation_state = 10;
}
```

**Device.* Event Protobuf Definitions:**

```protobuf
syntax = "proto3";

package coco.device.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";

// ===================================================================
// Device.Heartbeat - Full state snapshot (published every ~5-10s)
// ===================================================================

message DeviceHeartbeat {
  string device_name = 1;                          // "devices/{serial_number}"
  Device device = 2;                               // Full device state snapshot
  google.protobuf.Timestamp received_at = 3;       // When Device Service received the heartbeat
  google.protobuf.FieldMask updated_fields = 4;    // Which fields changed since last heartbeat
}

// ===================================================================
// Device.Moved - Location changed significantly (> 10m)
// ===================================================================

message DeviceMoved {
  string device_name = 1;                          // "devices/{serial_number}"
  GpsLocation previous_location = 2;               // Location before movement
  GpsLocation new_location = 3;                    // Current location
  double distance_meters = 4;                      // Distance traveled since last event
  google.protobuf.Timestamp moved_at = 5;          // When movement was detected
}

message GpsLocation {
  double latitude = 1;
  double longitude = 2;
  double heading = 3;                              // Degrees from north (0-360)
  double horizontal_accuracy = 4;                  // Accuracy in meters
  double altitude = 5;                             // Meters above sea level
  double speed_mph = 6;                            // Current speed
}

// ===================================================================
// Device.LidOpened - Lid state changed from closed to open
// ===================================================================

message DeviceLidOpened {
  string device_name = 1;                          // "devices/{serial_number}"
  google.protobuf.Timestamp opened_at = 2;         // When lid opened
  LidOpenTrigger trigger = 3;                      // What caused the lid to open
  string request_id = 4;                           // If triggered by command, the request ID
}

enum LidOpenTrigger {
  LID_OPEN_TRIGGER_UNSPECIFIED = 0;
  LID_OPEN_TRIGGER_BUTTON = 1;                     // Physical button press
  LID_OPEN_TRIGGER_PIN = 2;                        // PIN entered and validated
  LID_OPEN_TRIGGER_COMMAND = 3;                    // Remote unlock command
  LID_OPEN_TRIGGER_MANUAL = 4;                     // Manual/forced open
}

// ===================================================================
// Device.LidClosed - Lid state changed from open to closed
// ===================================================================

message DeviceLidClosed {
  string device_name = 1;                          // "devices/{serial_number}"
  google.protobuf.Timestamp closed_at = 2;         // When lid closed
  int32 seconds_open = 3;                          // How long lid was open
  bool was_timeout = 4;                            // True if auto-closed due to timeout
}

// ===================================================================
// Device.LidJammed - Lid mechanism reports fault
// ===================================================================

message DeviceLidJammed {
  string device_name = 1;                          // "devices/{serial_number}"
  string fault_code = 2;                           // Specific lid fault code from hardware
  string message = 3;                              // Human-readable fault description
  google.protobuf.Timestamp detected_at = 4;       // When fault was detected
}

// ===================================================================
// Device.BatteryLow - Battery crossed 20% threshold (downward)
// ===================================================================

message DeviceBatteryLow {
  string device_name = 1;                          // "devices/{serial_number}"
  int32 battery_percent = 2;                       // Current battery percentage
  bool is_charging = 3;                            // Whether battery is currently charging
  int32 battery_index = 4;                         // Which battery (for multi-battery robots)
  google.protobuf.Timestamp detected_at = 5;       // When threshold was crossed
}

// ===================================================================
// Device.BatteryCritical - Battery crossed 5% threshold (downward)
// ===================================================================

message DeviceBatteryCritical {
  string device_name = 1;                          // "devices/{serial_number}"
  int32 battery_percent = 2;                       // Current battery percentage
  bool is_charging = 3;                            // Whether battery is currently charging
  int32 battery_index = 4;                         // Which battery (for multi-battery robots)
  google.protobuf.Timestamp detected_at = 5;       // When threshold was crossed
}

// ===================================================================
// Device.ConnectivityChanged - Online/offline transition
// ===================================================================

message DeviceConnectivityChanged {
  string device_name = 1;                          // "devices/{serial_number}"
  bool online = 2;                                 // New online status
  bool previous_online = 3;                        // Previous online status
  ConnectivitySource source = 4;                   // Which source triggered the change
  google.protobuf.Timestamp changed_at = 5;        // When connectivity changed
}

enum ConnectivitySource {
  CONNECTIVITY_SOURCE_UNSPECIFIED = 0;
  CONNECTIVITY_SOURCE_IOT = 1;                     // AWS IoT Core heartbeat timeout
  CONNECTIVITY_SOURCE_DRIVEU = 2;                  // DriveU streamer status
  CONNECTIVITY_SOURCE_AGGREGATED = 3;              // Combined from multiple sources
}

// ===================================================================
// Device.HealthDegraded - Component entered fault state
// ===================================================================

message DeviceHealthDegraded {
  string device_name = 1;                          // "devices/{serial_number}"
  string component_name = 2;                       // e.g., "GPS", "CAMERAS", "SEGWAY_BASE"
  HardwareStatusCode status_code = 3;              // Severity of the fault
  string fault_code = 4;                           // Specific fault code (e.g., "TIMEOUT", "CRC_ERROR")
  string message = 5;                              // Human-readable description
  google.protobuf.Timestamp detected_at = 6;       // When fault was detected
}

enum HardwareStatusCode {
  HARDWARE_STATUS_CODE_UNSPECIFIED = 0;
  HARDWARE_STATUS_CODE_OK = 1;                     // Component is working normally
  HARDWARE_STATUS_CODE_READY = 2;                  // Component is ready but not active
  HARDWARE_STATUS_CODE_TIMEOUT = 3;                // Component communication timeout
  HARDWARE_STATUS_CODE_NON_CRITICAL_FAULT = 4;     // Fault but robot can operate
  HARDWARE_STATUS_CODE_CRITICAL_FAULT = 5;         // Fault that grounds the robot
}

// ===================================================================
// Device.HealthRestored - Component returned to OK state
// ===================================================================

message DeviceHealthRestored {
  string device_name = 1;                          // "devices/{serial_number}"
  string component_name = 2;                       // Component that recovered
  int32 downtime_seconds = 3;                      // How long the component was degraded
  google.protobuf.Timestamp restored_at = 4;       // When component returned to OK
}

// ===================================================================
// Device.EmergencyStop - Emergency stop triggered
// ===================================================================

message DeviceEmergencyStop {
  string device_name = 1;                          // "devices/{serial_number}"
  EmergencyStopSource source = 2;                  // What triggered the emergency stop
  string triggered_by = 3;                         // User ID if remote, null if physical
  google.protobuf.Timestamp triggered_at = 4;      // When emergency stop was triggered
}

enum EmergencyStopSource {
  EMERGENCY_STOP_SOURCE_UNSPECIFIED = 0;
  EMERGENCY_STOP_SOURCE_BUTTON = 1;                // Physical button on robot
  EMERGENCY_STOP_SOURCE_REMOTE = 2;                // Sent from Mission Control
  EMERGENCY_STOP_SOURCE_SYSTEM = 3;                // Automatic (e.g., detected collision)
}

// ===================================================================
// Device.PinEntry - Key pressed on device pin pad
// ===================================================================

message DevicePinEntry {
  string device_name = 1;                          // "devices/{serial_number}"
  string entered_key = 2;                          // The key that was entered (single digit/char)
  google.protobuf.Timestamp entered_at = 3;        // When key was pressed
}

// ===================================================================
// Device.CargoChanged - Cargo presence state changed
// ===================================================================

message DeviceCargoChanged {
  string device_name = 1;                          // "devices/{serial_number}"
  bool has_cargo = 2;                              // New cargo presence state
  bool previous_has_cargo = 3;                     // Previous cargo presence state
  google.protobuf.Timestamp changed_at = 4;        // When cargo state changed
}

// ===================================================================
// Supporting message: Device (full snapshot)
// ===================================================================

message Device {
  string name = 1;                                 // "devices/{serial_number}"
  GpsLocation location = 10;                       // Current GPS location
  repeated Battery batteries = 11;                 // Battery states
  Connectivity connectivity = 12;                  // Online/offline status
  LidState lid_state = 13;                         // Current lid state
  repeated ComponentStatus component_statuses = 20; // Component health
  DriveUStatus driveu_status = 21;                 // Teleoperation status
  bool has_cargo = 22;                             // Cargo presence
  google.protobuf.Timestamp created_at = 100;
  google.protobuf.Timestamp updated_at = 101;
}

message Battery {
  int32 battery_index = 1;
  int32 charge_percent = 2;
  bool is_charging = 3;
  double voltage = 4;
  double current = 5;
  double temperature = 6;
  google.protobuf.Timestamp updated_at = 8;
}

message Connectivity {
  bool online = 1;
  NetworkType network_type = 2;
  int32 signal_strength = 3;
  google.protobuf.Timestamp updated_at = 4;
}

enum NetworkType {
  NETWORK_TYPE_UNSPECIFIED = 0;
  NETWORK_TYPE_WIFI = 1;
  NETWORK_TYPE_CELLULAR = 2;
  NETWORK_TYPE_ETHERNET = 3;
}

message LidState {
  bool is_open = 1;
  google.protobuf.Timestamp updated_at = 2;
}

message ComponentStatus {
  string component_name = 1;
  HardwareStatusCode status_code = 2;
  string message = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message DriveUStatus {
  DriveUStreamerStatus streamer_status = 1;
  bool pilot_connected = 2;
  string session_id = 3;
  google.protobuf.Timestamp updated_at = 4;
}

enum DriveUStreamerStatus {
  DRIVEU_STREAMER_STATUS_UNSPECIFIED = 0;
  DRIVEU_STREAMER_STATUS_OFFLINE = 1;
  DRIVEU_STREAMER_STATUS_ONLINE = 2;
  DRIVEU_STREAMER_STATUS_CONNECTED_TO_NODE = 3;
  DRIVEU_STREAMER_STATUS_STREAMING = 4;
}
```

---

### Lid Service (renamed from State Service)

The current State Service becomes the **Lid Service**, focused solely on lid-related functionality.

**What it owns:**
- Lid cycles (open/close tracking)
- PIN entry flows
- Lid timeout management

**What it does NOT do (moved to other services):**
- Telemetry processing → Device Service
- Connectivity tracking → Device Service
- Operation state machine → Operations Service (replaced by issues)

**Events Published:**

| Event | Trigger | Purpose |
|-------|---------|---------|
| `LidCycle.Init` | Lid opened | Track loading/unloading started |
| `LidCycle.Complete` | Lid closed | Track loading/unloading completed |
| `LidCycle.Timeout` | Lid open too long | Alert for stuck lid |
| `Robots.PinEntry` | PIN entered on keypad | Validate PIN, trigger lid action |

---

### Device Issues Module (NEW - within Operations Service)

This is a new module within Operations Service, not a separate service.

```
service/operations/src/modules/
├── device-issues/              # NEW MODULE
│   ├── device-issues.module.ts
│   ├── device-issues.service.ts
│   ├── device-issues.repository.ts
│   ├── device-issues.handler.ts      # Consumes Device.HealthDegraded
│   └── definitions/
│       ├── issue-types.ts
│       ├── dtos/
│       └── events.ts
├── fo-tasks/                   # Existing - can create FO tasks from issues
├── fo-requests/                # Existing - now links to issues
└── trips/                      # Existing
```

**What it owns:**
- Device issue lifecycle (raised → active → cleared)
- Operator-raised issues
- Association with FO assistance requests

**What it does:**
- Consumes `Device.HealthDegraded` events
- Creates issues (with optional FO task creation in same transaction)
- Can trigger FO task creation (via existing FO Tasks module)

**Internal Events (within Operations Service):**

| Event | Trigger | Purpose |
|-------|---------|---------|
| `DeviceIssue.Raised` | New issue created | Triggers state deriver to recompute grounded status |
| `DeviceIssue.Cleared` | Issue resolved (operator or auto) | Triggers state deriver to recompute grounded status |

#### Issue Types and Severity

```typescript
enum IssueSeverity {
  WARNING = 'WARNING',     // Does NOT ground robot
  CRITICAL = 'CRITICAL',   // Grounds robot
  SAFETY = 'SAFETY',       // Grounds robot + immediate alerts
}

// Predefined issue types (hardcoded initially, configurable later if needed)
const ISSUE_TYPES = {
  // Device health issues (created from Device.HealthDegraded events)
  GPS_FAULT: { severity: 'CRITICAL', grounds: true },
  LTE_TIMEOUT: { severity: 'WARNING', grounds: false },
  CAMERA_FAULT: { severity: 'CRITICAL', grounds: true },
  SEGWAY_FAULT: { severity: 'SAFETY', grounds: true },
  
  // Operator-raised issues
  MANUAL_GROUND: { severity: 'CRITICAL', grounds: true },
  SUSPICIOUS_BEHAVIOR: { severity: 'WARNING', grounds: false },
  UNDER_INVESTIGATION: { severity: 'CRITICAL', grounds: true },
  
  // FO Assistance issues (linked to FoAssistanceRequest)
  FO_ASSISTANCE_REQUESTED: { severity: 'CRITICAL', grounds: true },
};
```

#### Issue Lifecycle

```
                           ┌────────────┐
     Device.HealthDegraded │            │ Operator raises
     or Operator action    │   RAISED   │ via API
            ┌──────────────┤  (active)  ├──────────────┐
            │              │            │              │
            ▼              └─────┬──────┘              ▼
                                 │            ┌───────────────┐
                                 │            │   ACTIVE      │
                                 │            │               │
                                 │            └───────┬───────┘
                                 │                    │
                    ┌────────────┼────────────────────┼────────────┐
                    │            │                    │            │
                    ▼            ▼                    ▼            ▼
            ┌─────────────┐            ┌─────────────┐ ┌─────────────┐
            │   CLEARED   │            │ AUTO_CLEARED│ │   CLEARED   │
            │ (by health) │            │ (by health) │ │(by operator)│
            └─────────────┘            └─────────────┘ └─────────────┘
```

#### FoAssistanceRequest Integration

`FoAssistanceRequest` records are now **associated with issues**. When an FO assistance request with `needsGrounding = true` is created, a corresponding issue is also created:

```typescript
// When FoAssistanceRequest is created
async function createFoAssistanceRequest(data: CreateFoRequestDto) {
  const foRequest = await foRequestRepo.create(data);
  
  // If grounding is needed, create associated issue
  if (data.needsGrounding) {
    const issue = await deviceIssuesService.raiseIssue({
      deviceSerial: data.robotSerial,
      type: 'FO_ASSISTANCE_REQUESTED',
      reason: data.notes,
      foAssistanceRequestId: foRequest.id,  // Link back
    });
    
    // Update FO request with issue reference
    await foRequestRepo.update(foRequest.id, { issueId: issue.id });
  }
  
  return foRequest;
}
```

**Schema change for FoAssistanceRequest:**

```sql
ALTER TABLE fo_assistance_requests 
ADD COLUMN issue_id UUID REFERENCES device_issues(id);
```

This allows:
- Grounding to be issue-based (consistent mechanism)
- FO request context preserved (why was this issue created?)
- Future migration: FO requests could become a type of issue

---

### Operations Service (Enhanced)

**What it owns:**
- Trips lifecycle
- Deployments lifecycle
- FO Tasks lifecycle
- FO Requests lifecycle (now with issue association)
- **Device Issues lifecycle** (via new Device Issues module)
- **Derived operational state** (all computed, not stored)

**What it computes (from source tables):**

| Field | Source | Derivation |
|-------|--------|------------|
| `deployment_status` | `RobotDeployment` | `deployment.isActive ? 'DEPLOYED' : 'UNDEPLOYED'` |
| `trip_status` | `Trip` | `activeTrip ? 'ON_TRIP' : 'IDLE'` |
| `grounded` | `device_issues` | `issues.some(i => i.grounds_robot)` |
| `needs_maintenance` | `device_issues` + `FoTaskModel` | `issues.some(i => i.severity === 'WARNING') \|\| foTasks.some(...)` |
| `undergoing_maintenance` | `FoTaskModel` | `foTasks.some(t => t.state === 'IN_PROGRESS')` |
| `operation_state` | Derived | Priority: GROUNDED > ON_TRIP > PARKED > OFF_DUTY |

**Events Published:**

| Event | Trigger | Payload | Purpose |
|-------|---------|---------|---------|
| `Robot.OperationalStateChanged` | Any derived state change | `serial`, `deployment_status`, `grounded`, `on_trip`, `operation_state` (legacy) | Notify downstream services |

**State Computation Flow:**

```typescript
// Called whenever deployment, trip, issue, or FO task state changes
async function recomputeAndPublishState(serial: string) {
  const state = await deriveOperationalState(serial);
  
  // Compute behavior fields
  const lightMode = deriveLightMode(state);
  const soundProfile = deriveSoundProfile(state);
  const otaEnabled = deriveOtaEnabled(state);
  
  // Write to shadow via Device Service
  await deviceService.setShadowFields(serial, {
    operation_state: state.operation_state,  // Legacy
    light_mode: lightMode,
    sound_profile: soundProfile,
    ota_enabled: otaEnabled,
  });
  
  // Publish for downstream consumers
  await publish('Robot.OperationalStateChanged', {
    serial,
    deployment_status: state.deployment_status,
    trip_status: state.trip_status,
    grounded: state.grounded,
    needs_maintenance: state.needs_maintenance,
    operation_state: state.operation_state,  // Legacy
  });
}
```

---

## RobotStateHistory Simplification

### Current State

`RobotStateHistory` currently stores many fields that are forwarded via `Robots.StateChange`:
- `operationState`
- `needsMaintenance`
- `undergoingMaintenance`
- `hasFood`
- `needsMovement`
- `needsPickup`
- `tripType`
- `driveable` (never actually set - always null)

### New Approach: Audit Log Only

`RobotStateHistory` becomes an **audit log** for tracking state changes over time. It is **NOT** used to derive current state.

**Fields to keep (for audit):**
- `operationState` - what state was the robot in
- `hasFood` → replaced by `hasCargo` from device
- `tripType` - what kind of trip
- `createdAt` - when did this happen

**Fields to remove:**
- `driveable` - never used
- `needsMaintenance` - now derived from issues
- `needsMovement` - now derived from trip status
- `undergoingMaintenance` - now derived from FO tasks
- `needsPickup` - now derived from FO tasks

**Usage change:**
- **Before**: `getLatestRobotStateHistory()` to check `needsMaintenance`
- **After**: `deriveOperationalState()` which queries source tables

---

## hasCargo: Replacing hasFood

A parallel effort is adding `hasCargo` as a device-reported field, which replaces the current `hasFood` inference from lid events.

**Current state:**
- `hasFood` is set by Deliveries Service when robot is loaded
- Requires inference from lid close events

**New state:**
- Robot publishes `hasCargo: boolean` in heartbeat
- Device Service publishes `Device.CargoChanged` event
- Operations Service (and others) consume this event

**Benefits:**
- More reliable (robot knows if cargo is present)
- Simpler event flow (no inference needed)
- Works for non-food cargo in future

---

## Field Ownership & Derivation Table

| Field | Owner | Source/Derivation | Stored Where |
|-------|-------|-------------------|--------------|
| **Physical State (Facts)** ||||
| `location` | Device Service | MQTT heartbeat | `devices.location` |
| `battery_percent` | Device Service | MQTT heartbeat | `devices.batteries` |
| `connectivity.online` | Device Service | IoT Core + DriveU | `devices.online` |
| `lid_is_open` | Lid Service | MQTT events | `lid_cycles` |
| `component_statuses[]` | Device Service | MQTT heartbeat | `device_components` |
| `has_cargo` | Device Service | MQTT heartbeat (new) | `devices.has_cargo` |
| **Issue State** ||||
| `active_issues[]` | Operations Service | Device events + operators | `device_issues` table |
| **Deployment State** ||||
| `active_deployment` | Operations Service | Deployment lifecycle | `robot_deployments` table |
| `active_trip` | Operations Service | Trip lifecycle | `trips` table |
| **Derived State (NOT stored)** ||||
| `deployment_status` | Operations Service | `deployment.isActive` | Computed |
| `trip_status` | Operations Service | `trip != null` | Computed |
| `grounded` | Operations Service | `issues.some(i => i.grounds_robot)` | Computed |
| `needs_maintenance` | Operations Service | `issues.some(warning) \|\| foTasks.pending` | Computed |
| `undergoing_maintenance` | Operations Service | `foTasks.some(in_progress)` | Computed |
| **Shadow Desired (Written by Operations)** ||||
| `operation_state` | Operations Service | **DEPRECATED** - computed for backward compat | Device shadow |
| `light_mode` | Operations Service | Computed from operational state | Device shadow |
| `sound_profile` | Operations Service | Computed from operational state | Device shadow |
| `ota_enabled` | Operations Service | Computed from operational state | Device shadow |

---

## Device Issue Data Model

```sql
-- Device Issues table (in Operations Service database)
CREATE TABLE device_issues (
    id UUID PRIMARY KEY,
    device_serial VARCHAR(255) NOT NULL,
    
    -- Issue identification
    issue_type VARCHAR(100) NOT NULL,
    source VARCHAR(50) NOT NULL,       -- DEVICE_HEALTH, OPERATOR, FO_REQUEST
    severity VARCHAR(20) NOT NULL,     -- WARNING, CRITICAL, SAFETY
    
    -- Grounding
    grounds_robot BOOLEAN NOT NULL,
    
    -- State
    status VARCHAR(20) NOT NULL,       -- ACTIVE, CLEARED, AUTO_CLEARED
    
    -- Content
    title VARCHAR(255) NOT NULL,
    description TEXT,
    reason TEXT,                       -- For MANUAL_GROUND
    component_name VARCHAR(100),       -- For device health issues
    fault_code VARCHAR(50),
    
    -- Links
    fo_assistance_request_id UUID,     -- Link to FoAssistanceRequest if applicable
    
    -- Audit
    raised_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    raised_by VARCHAR(255),            -- user_id or 'system'
    cleared_at TIMESTAMPTZ,
    cleared_by VARCHAR(255),
    clear_reason TEXT,
    
    CONSTRAINT issues_one_active_per_type UNIQUE (device_serial, issue_type) 
        WHERE status = 'ACTIVE'
);

CREATE INDEX idx_issues_device_active ON device_issues(device_serial) 
    WHERE status = 'ACTIVE';
```

---

## API Design

### Operations Service API (includes Device Issues)

```protobuf
service OperationsService {
  // Deployment management
  rpc CreateDeployment(CreateDeploymentRequest) returns (Deployment);
  rpc EndDeployment(EndDeploymentRequest) returns (Deployment);
  
  // Device Issue management (via Device Issues module)
  rpc RaiseDeviceIssue(RaiseDeviceIssueRequest) returns (DeviceIssue);
  rpc ClearDeviceIssue(ClearDeviceIssueRequest) returns (DeviceIssue);
  rpc GetDeviceIssue(GetDeviceIssueRequest) returns (DeviceIssue);
  rpc ListDeviceIssues(ListDeviceIssuesRequest) returns (ListDeviceIssuesResponse);
  
  // Convenience methods (semantic actions)
  rpc GroundRobot(GroundRobotRequest) returns (DeviceIssue);  // Creates MANUAL_GROUND issue
  rpc UngroundRobot(UngroundRobotRequest) returns (UngroundRobotResponse);  // Clears grounding issues
  
  // Status queries (all derived, not stored)
  rpc GetRobotOperationalState(GetRobotOperationalStateRequest) 
      returns (RobotOperationalState);
}

message RaiseDeviceIssueRequest {
  string device_serial = 1;
  string issue_type = 2;              // e.g., "MANUAL_GROUND", "GPS_FAULT"
  optional string reason = 3;         // For operator-raised issues
  optional string description = 4;
}

message ClearDeviceIssueRequest {
  string issue_id = 1;
  string clear_reason = 2;
}

message GroundRobotRequest {
  string device_serial = 1;
  string reason = 2;
  optional string description = 3;
}

message UngroundRobotRequest {
  string device_serial = 1;
  string clear_reason = 2;
  optional bool clear_all_issues = 3;  // Clear all grounding issues
  optional string issue_id = 4;        // Or clear specific issue
}

message RobotOperationalState {
  string device_serial = 1;
  DeploymentStatus deployment_status = 2;
  TripStatus trip_status = 3;
  bool grounded = 4;
  bool needs_maintenance = 5;
  bool undergoing_maintenance = 6;
  bool has_cargo = 7;
  
  // Active issues summary
  repeated IssueSummary active_issues = 10;
  
  // Legacy (deprecated)
  string operation_state = 20;
}
```

---

## UI Migration: Semantic Actions

### Current State: Low-Level Transitions

MRO and Mission Control currently use:

```typescript
// Current: Direct state transition
POST /state/{serial}/request-transition
{
  "fromStates": ["PARKED"],
  "toState": "GROUNDED",
  "comment": "reason: Suspicious behavior, comment: Robot drove erratically"
}
```

### Target State: Semantic Actions

| Current UI Action   | Old API                                                              | New API                                                                                | Service    |
| ------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ---------- |
| Ground robot        | `POST /state/{serial}/request-transition {toState: GROUNDED}`        | `POST /operations/device-issues` with `{type: 'MANUAL_GROUND'}`                        | Operations |
| Unground robot      | `POST /state/{serial}/request-transition {toState: PARKED/OFF_DUTY}` | `DELETE /operations/device-issues/{id}` or `POST /operations/robots/{serial}/unground` | Operations |
| Deploy robot        | `POST /state/{serial}/request-transition {toState: PARKED}`          | `POST /operations/deployments`                                                         | Operations |
| Undeploy robot      | `POST /state/{serial}/request-transition {toState: OFF_DUTY}`        | `DELETE /operations/deployments/{id}`                                                  | Operations |
| View issues         | N/A                                                                  | `GET /operations/device-issues?device_serial={serial}`                                 | Operations |

### New MRO UI Mockup

```
┌─────────────────────────────────────────────────────────────────┐
│  Robot C1-0042                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Status                                                          │
│  ─────────────────────────────────────────────────────────────── │
│  Deployment: DEPLOYED at Parking Lot A                          │
│  Operational: ⚠️ GROUNDED (1 active issue)                       │
│  Cargo: Empty                                                    │
│                                                                  │
│  Active Issues                                                   │
│  ─────────────────────────────────────────────────────────────── │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 🔴 MANUAL_GROUND                           [Clear Issue]    ││
│  │ "Robot drove erratically during last trip"                  ││
│  │ Raised by operator@coco.co • 2 hours ago                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Actions                                                         │
│  ─────────────────────────────────────────────────────────────── │
│  [Ground Robot]  [Report Issue]  [End Deployment]               │
│                                                                  │
│  Issue History                                             [▼]  │
│  ─────────────────────────────────────────────────────────────── │
│  • GPS_FAULT - Auto-cleared after 5 min (yesterday)            │
│  • MANUAL_GROUND - Cleared by tech@coco.co (3 days ago)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Backward Compatibility

### Robot Firmware

Current robot firmware reads `operation_state` from the shadow to control:
- Lights (different patterns per state)
- Sounds (different profiles per state)
- OTA permissions (only in OFF_DUTY)

**Strategy: Dual-write during transition**

```typescript
// Operations Service computes both old and new fields
await deviceService.setShadowFields(serial, {
  // Legacy (robot reads this today)
  operation_state: 'GROUNDED',
  
  // New explicit fields (robot will read these after firmware update)
  light_mode: 'MAINTENANCE',
  sound_profile: 'SILENT',
  ota_enabled: false,
});
```

### Event Consumers

Services currently consuming `Robots.StateChange`:

| Consumer           | Current Field    | New Event                       | New Field                       |
| ------------------ | ---------------- | ------------------------------- | ------------------------------- |
| Dispatch Engine    | `operationState` | `Robot.OperationalStateChanged` | `grounded`, `deployment_status` |
| Fleet Service (Go) | `operationState` | `Robot.OperationalStateChanged` | `grounded`, `deployment_status` |
| Operations Service | `operationState` | Internal (self-publishes)       | N/A                             |

**Strategy: Publish both events during transition**

```typescript
// Operations Service publishes both
await publish('Robots.StateChange', { serial, operationState: 'GROUNDED', ... });
await publish('Robot.OperationalStateChanged', { serial, grounded: true, ... });
```

---

## Event Flow Examples

### Example 1: GPS Fault Grounds Robot

```
Timeline:
─────────────────────────────────────────────────────────────────────────────
1. Robot reports GPS_TIMEOUT in heartbeat
   Device Service → publishes Device.HealthDegraded(component: GPS)

2. Operations Service (Device Issues module) receives event
   - Creates issue (GPS_FAULT, grounds: true)
   - Triggers Operational State Deriver

3. Operational State Deriver recomputes state
   - grounded = true (derived from active issues)
   - operation_state = GROUNDED
   - Writes shadow: { operation_state: GROUNDED, light_mode: MAINTENANCE }
   - Publishes: Robot.OperationalStateChanged

4. Robot reads shadow, switches to maintenance lights
─────────────────────────────────────────────────────────────────────────────
```

### Example 2: Operator Grounds Robot

```
Timeline:
─────────────────────────────────────────────────────────────────────────────
1. Operator clicks "Ground Robot" in MRO
   MRO → POST /operations/device-issues { type: MANUAL_GROUND, reason: "..." }

2. Operations Service (Device Issues module) creates issue
   - Triggers Operational State Deriver

3. Operational State Deriver recomputes state
   - grounded = true
   - Writes shadow, publishes event (same as above)
─────────────────────────────────────────────────────────────────────────────
```

### Example 3: FO Assistance Request with Grounding

```
Timeline:
─────────────────────────────────────────────────────────────────────────────
1. Pilot requests FO assistance with needsGrounding=true
   Operations Service creates FoAssistanceRequest

2. Device Issues module creates associated issue
   - type: FO_ASSISTANCE_REQUESTED
   - fo_assistance_request_id: <request_id>
   - grounds: true

3. Operational State Deriver recomputes state
   - grounded = true
   - Writes shadow, publishes event

4. When FO resolves the issue:
   - Clearing the issue also marks FO request as resolved
   - grounded = false (automatically)
─────────────────────────────────────────────────────────────────────────────
```

---

## Work Breakdown

### Chunk 1: Device Service Foundation
- Create new Device Service
- Move telemetry processing from State Service
- Move connectivity handling from State Service
- Implement `Device.Heartbeat`, `Device.HealthDegraded`, `Device.HealthRestored` events
- Implement `SetShadowFields` RPC

### Chunk 2: Device Issues Module
- Create `device_issues` table in Operations DB
- Implement issue CRUD operations
- Implement `RaiseDeviceIssue`, `ClearDeviceIssue` RPCs
- Implement `GroundRobot`, `UngroundRobot` convenience RPCs
- Add handler for `Device.HealthDegraded` events

### Chunk 3: FoAssistanceRequest Integration
- Add `issue_id` column to `fo_assistance_requests` table
- Update FO request creation to create associated issue when `needsGrounding=true`
- Update FO request resolution to clear associated issue

### Chunk 4: Operational State Deriver
- Implement `deriveOperationalState()` function
- Replace `getLatestRobotStateHistory()` usages with derived state
- Implement `Robot.OperationalStateChanged` event publishing
- Wire up triggers: deployment changes, trip changes, issue changes, FO task changes

### Chunk 5: Lid Service Extraction
- Rename State Service to Lid Service
- Remove operation state logic
- Remove telemetry processing (moved to Device Service)
- Keep only lid cycle and PIN entry functionality

### Chunk 6: RobotStateHistory Simplification
- Remove unused `driveable` field
- Update `RobotStateHistory` to be audit-only
- Remove usages that derive current state from history
- Keep history writes for audit trail

### Chunk 7: UI Migration
- Update MRO to use semantic actions (issue-based grounding)
- Update Mission Control to use semantic actions
- Add issue list views to device detail pages
- Remove StateTransition component usage

### Chunk 8: Event Consumer Migration
- Update Dispatch Engine to consume `Robot.OperationalStateChanged`
- Update Fleet Service to consume `Robot.OperationalStateChanged`
- Dual-publish `Robots.StateChange` + `Robot.OperationalStateChanged` during transition

### Chunk 9: hasCargo Integration
- Update Device Service to process `hasCargo` from heartbeat
- Publish `Device.CargoChanged` event
- Update consumers to use `hasCargo` instead of `hasFood`

### Future Work (Deferred)
- **Debouncing/Hysteresis**: Add configurable debounce rules for issue creation from health events
- **Issue Suppressions**: Add ability to suppress issue creation for specific types/devices
- **TTL Auto-clear**: Add time-based automatic issue clearing
- **Auto-rules**: Automated issue creation based on configurable rules

---

## Open Questions

1. **Issue type taxonomy**: Should issue types be hardcoded or configurable? Recommend starting hardcoded, make configurable if needed.

2. **Audit retention**: How long to keep cleared issue history? Recommend 90 days in hot storage, archive to cold storage after.

3. **Severity override**: Can operators upgrade/downgrade issue severity? Recommend no initially—clear and re-raise if needed.

---

## Appendix: Device.* Event Consumer Reference

This section provides a comprehensive breakdown of all `Device.*` events published by the Device Service, including their consumers and what each consumer does with the event.

### Device.Heartbeat

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Dispatch Engine** | `eventsink/handlers/device-heartbeat.handler.ts` | Update robot supply cache for assignment decisions | `location`, `battery_percent`, `online`, `component_statuses` |
| **Fleet Service (Go)** | `fleet/internal/consumer/device_consumer.go` | Update Redis cache for Uber position updates, DynamoDB for persistence | `location`, `battery_percent`, `online`, `has_cargo` |
| **Trip Monitor** | `trip-monitor/handlers/device-heartbeat.handler.ts` | Update trip trace, recalculate ETA | `location`, `updated_at` |
| **Operations Service** | `operations/handlers/device-heartbeat.handler.ts` | Update Redis ephemeral state | `location`, `online`, `component_statuses` |

### Device.Moved

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Trip Monitor** | `trip-monitor/handlers/device-moved.handler.ts` | Route deviation detection, arrival detection | `new_location`, `previous_location`, `distance_meters` |
| **Operations Service** | `operations/handlers/device-moved.handler.ts` | Geofence checks (entering/leaving service areas) | `new_location`, `device_name` |

### Device.LidOpened

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Lid Service** | `lid-service/handlers/lid-opened.handler.ts` | Start lid cycle, track open duration | `device_name`, `opened_at`, `trigger`, `request_id` |
| **Deliveries Service** | `deliveries/handlers/lid-opened.handler.ts` | Track loading start time for active deliveries | `device_name`, `opened_at` |

### Device.LidClosed

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Lid Service** | `lid-service/handlers/lid-closed.handler.ts` | Complete lid cycle, emit `LidCycle.Completed` | `device_name`, `closed_at`, `seconds_open`, `was_timeout` |
| **Deliveries Service** | `deliveries/handlers/lid-closed.handler.ts` | Trigger load completion if conditions met | `device_name`, `closed_at`, `seconds_open` |

### Device.LidJammed

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Lid Service** | `lid-service/handlers/lid-jammed.handler.ts` | Abort lid cycle, alert operations | `device_name`, `fault_code` |
| **Operations Service** | `operations/handlers/device-issues/lid-jammed.handler.ts` | Create urgent FO task | `device_name`, `fault_code`, `message` |
| **Deliveries Service** | `deliveries/handlers/lid-jammed.handler.ts` | Handle failed handoff scenario | `device_name` |

### Device.BatteryLow

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** | `operations/handlers/device-issues/battery-low.handler.ts` | Create FO task for battery swap (non-urgent) | `device_name`, `battery_percent`, `is_charging` |
| **Dispatch Engine** | `dispatch-engine/handlers/battery-low.handler.ts` | Reduce robot availability score | `device_name`, `battery_percent` |

### Device.BatteryCritical

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** | `operations/handlers/device-issues/battery-critical.handler.ts` | Raise CRITICAL issue (grounds robot), create urgent FO task | `device_name`, `battery_percent`, `is_charging` |
| **Dispatch Engine** | `dispatch-engine/handlers/battery-critical.handler.ts` | Remove from available supply | `device_name` |
| **Trip Monitor** | `trip-monitor/handlers/battery-critical.handler.ts` | Alert if robot is mid-trip | `device_name`, `battery_percent` |

### Device.ConnectivityChanged

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** | `operations/handlers/device-connectivity.handler.ts` | Update robot connectivity status, potentially raise issue if offline too long | `device_name`, `online`, `previous_online`, `changed_at` |
| **Dispatch Engine** | `dispatch-engine/handlers/device-connectivity.handler.ts` | Remove/add to supply based on online status | `device_name`, `online` |

### Device.HealthDegraded

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** (Device Issues module) | `operations/modules/device-issues/health-degraded.handler.ts` | Create issue based on component severity, potentially ground robot | `device_name`, `component_name`, `status_code`, `fault_code`, `message` |
| **Dispatch Engine** | `dispatch-engine/handlers/health-degraded.handler.ts` | Mark robot unhealthy if CRITICAL, remove from supply | `device_name`, `status_code` |

### Device.HealthRestored

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** (Device Issues module) | `operations/modules/device-issues/health-restored.handler.ts` | Auto-clear issue if applicable | `device_name`, `component_name`, `downtime_seconds` |
| **Dispatch Engine** | `dispatch-engine/handlers/health-restored.handler.ts` | Mark robot healthy if all components OK, add to supply | `device_name`, `component_name` |

### Device.EmergencyStop

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** | `operations/handlers/device-issues/emergency-stop.handler.ts` | Immediately raise SAFETY issue (grounds robot), create critical alert | `device_name`, `source`, `triggered_by`, `triggered_at` |
| **Dispatch Engine** | `dispatch-engine/handlers/emergency-stop.handler.ts` | Remove from supply immediately | `device_name` |
| **Trip Monitor** | `trip-monitor/handlers/emergency-stop.handler.ts` | Pause trip, alert if mid-delivery | `device_name` |

### Device.PinEntry

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Lid Service** | `lid-service/handlers/pin-entry.handler.ts` | Validate PIN against active unlock context, decide whether to open lid | `device_name`, `entered_key`, `entered_at` |

### Device.CargoChanged

| Consumer | Handler Location | Purpose | Fields Used |
|----------|------------------|---------|-------------|
| **Operations Service** | `operations/handlers/cargo-changed.handler.ts` | Update cargo status for trip tracking | `device_name`, `has_cargo`, `previous_has_cargo` |
| **Deliveries Service** | `deliveries/handlers/cargo-changed.handler.ts` | Update delivery state (loaded/unloaded) | `device_name`, `has_cargo` |
| **Dispatch Engine** | `dispatch-engine/handlers/cargo-changed.handler.ts` | Update supply cache with cargo status | `device_name`, `has_cargo` |

### Event Routing Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Device Service                                       │
│                                                                             │
│  Publishes to RabbitMQ exchange: device.events                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RabbitMQ (Fan-out Exchange)                            │
│                                                                             │
│  Routing keys:                                                              │
│  - device.heartbeat.{serial}         → Device.Heartbeat                     │
│  - device.moved.{serial}             → Device.Moved                         │
│  - device.lid.opened.{serial}        → Device.LidOpened                     │
│  - device.lid.closed.{serial}        → Device.LidClosed                     │
│  - device.lid.jammed.{serial}        → Device.LidJammed                     │
│  - device.battery.low.{serial}       → Device.BatteryLow                    │
│  - device.battery.critical.{serial}  → Device.BatteryCritical               │
│  - device.connectivity.{serial}      → Device.ConnectivityChanged           │
│  - device.health.degraded.{serial}   → Device.HealthDegraded                │
│  - device.health.restored.{serial}   → Device.HealthRestored                │
│  - device.emergency.{serial}         → Device.EmergencyStop                 │
│  - device.pin.{serial}               → Device.PinEntry                      │
│  - device.cargo.{serial}             → Device.CargoChanged                  │
└─────────────────────────────────────────────────────────────────────────────┘
         │           │           │           │           │
         ▼           ▼           ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Ops Svc │ │Dispatch │ │  Fleet  │ │  Trip   │ │   Lid   │
    │         │ │ Engine  │ │ Service │ │ Monitor │ │ Service │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

---

## Appendix: Comparison with v2

| Aspect | v2 Design | v3 Design |
|--------|-----------|-----------|
| Grounding mechanism | State machine transition | Derived from active issues |
| Issue ownership | Not defined | Operations Service (Device Issues module) |
| Deployment vs grounded | Conflated in operation_state | Three orthogonal dimensions |
| Fleet Device Service | Read-only aggregator | Removed (Operations Service handles derivation) |
| Shadow writes | Device Service decides | Operations Service decides, Device Service writes |
| operator_hold | Separate table | MANUAL_GROUND issue type |
| UI actions | State transitions | Semantic actions (raise issue, clear issue, deploy, etc.) |
| FoAssistanceRequest | Independent | Associated with issue |
| RobotStateHistory | Source of current state | Audit log only |
| needsMaintenance | Stored flag | Derived from issues + FO tasks |
| hasFood | Inferred from lid events | `hasCargo` from device |
| driveable | Stored (but never set) | Removed |
