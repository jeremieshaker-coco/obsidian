# Device API Research Notes

## Executive Summary

Multiple services independently process MQTT/IoT events from robots, leading to:
- **Disparate state representations** across services
- **Varying data freshness** due to different processing times and caching strategies
- **No single source of truth** for device state
- **Inconsistent schemas** for similar device information
- **Error-prone** due to different event processing logic and potential message drops

## Event Flow Architecture

### 1. Event Ingestion
- **Source**: Robots send MQTT messages via AWS IoT
- **Queues**: Events are routed to two SQS queues:
  - `eventsQueueUrl` - Robot events (lid events, pin entry events)
  - `stateQueueUrl` - Heartbeat/state messages
- **Processor**: `IotStreamerService` (`service/state/src/iot-streamer/iot-streamer.service.ts`)

### 2. Event Processing
The `IotStreamerService` processes events and publishes to RabbitMQ:
- **Heartbeat Exchange**: `IoT.Heartbeat` - Contains battery, location, health, components, operationState
- **Health State Exchange**: `IoT.HealthStateChanged` - Published when health status changes
- **Lid Cycle Events**: Various lid-related events
- **Pin Entry Events**: `Robots.PinEntry`, `Robots.LidOpen`, `Robots.LidClose`

### 3. Event Consumers

#### State Service (`service/state`)
- **Role**: Primary event processor and state tracker
- **Storage**: 
  - `stateHistory` table (Prisma) - Stores robot state transitions
  - DynamoDB (via `StateTrackerService`) - Stores latest DriveU status
- **Publishes**: Heartbeat events, health state changes, lid cycle events
- **Key Files**:
  - `iot-streamer.service.ts` - Main event processor
  - `state.service.ts` - State management and queries
  - `heartbeat.service.ts` - Aggregates and stores heartbeats

#### Operations Service (`service/operations`)
- **Role**: Business logic and robot state management
- **Consumes**: 
  - `IoT.Heartbeat` - Updates ephemeral data and saves heartbeats
  - `Robots.StateChange` - Updates robot state history
- **Storage**:
  - `RobotStateHistory` table (Prisma) - Historical state changes
  - Redis cache - Ephemeral device info (location, battery, health, state)
  - DynamoDB (via `RobotEphemeralDataService`) - Latest device info
- **Key Files**:
  - `iot-heartbeat-handler.ts` - Processes heartbeats
  - `robot-state-change-handler.ts` - Processes state changes
  - `robot-ephemeral-data.service.ts` - Manages cached ephemeral data
  - `robots.service.ts` - Robot business logic
- **APIs**:
  - `GET /robots/:serial/state` - Current robot state
  - `GET /robots/:serial/status` - Robot status
  - `GET /internal/robots` - Internal robot list
  - `GET /robots/:serial/location` - Robot location

#### Dispatch Engine (`service/dispatch-engine`)
- **Role**: Planning and resource allocation
- **Consumes**:
  - `IoT.Heartbeat` - Updates robot supply (battery, location, health, operationState)
  - `Robots.StateChange` - Updates robot supply attributes (needsMovement, hasFood, etc.)
- **Storage**:
  - `Resource` table (Prisma) - Stores Robot/Pilot as JSON in `data` field
  - `Robot` table (Prisma) - V2 robot representation with demand relationships
- **Key Files**:
  - `iot-heartbeat.handler.ts` - Maps heartbeats to supply events
  - `robot-stage-change.handler.ts` - Maps state changes to supply events
  - `supply.repo.ts` - Robot/Pilot data access
- **Data Model**: Uses `RobotSupply` type with fields like `battery`, `healthy`, `location`, `operationState`, `needsMovement`, `hasFood`, `tripType`, `needsMaintenance`, etc.

#### Trip Monitor (`service/trip-monitor`)
- **Role**: Trip tracking and monitoring
- **Consumes**: `IoT.Heartbeat` - Updates trip state from heartbeat data
- **Storage**: Trip state database

#### Fleet Service (`coco-services/fleet`)
- **Role**: Fleet management and provider integrations
- **Consumes**: 
  - Kafka messages (protobuf) - Physical robot telemetry
  - RabbitMQ messages - Operations system business logic
- **Storage**: DynamoDB (`FleetRobot` entities)
- **Data Model**: Stores `Healthy`, `OperationalState`, `Location`, `HasFood`, `NeedsMaintenance`

## Data Models & Schemas

### State Service Models
```typescript
// Heartbeat payload
{
  serial: string;
  createdAt: number;
  receivedAt: number;
  processedAt: number;
  battery?: { soc: number; isCharging: boolean; sn?: string };
  location?: { latitude: number; longitude: number; heading: number; errHorz: number; updatedAt: number };
  healthy: boolean;
  components: Record<string, Component>;
  operationState?: OperationState; // 'GROUNDED' | 'ON_TRIP' | 'PARKED' | 'OFF_DUTY'
}
```

### Operations Service Models
```typescript
// RobotStateHistory (Prisma)
{
  robotSerial: string;
  needsMovement: boolean;
  hasFood: boolean;
  tripType: TripType;
  operationState: RobotStateEventState; // ON_TRIP | PARKED | GROUNDED | OFF_DUTY | DEPLOYED
  needsMaintenance: boolean;
  needsPickup: boolean;
  driveable: boolean;
  undergoingMaintenance: boolean;
  createdAt: DateTime;
}

// Ephemeral Device Info (Redis/DynamoDB)
{
  serial: string;
  location: { lat: number; lng: number; timestamp: number };
  battery: { currentBatteryPercentage: number; timestamp: number };
  healthy: boolean;
  robotState: RobotState; // GROUNDED | ON_TRIP | PARKED | etc.
}
```

### Dispatch Engine Models
```typescript
// RobotSupply
{
  robotSerial: string;
  location: Location | null;
  battery: number | null;
  healthy: boolean | null;
  needsMovement: boolean | null;
  hasFood: boolean | null;
  tripType: TripType | null;
  needsMaintenance: boolean | null;
  needsPickup: boolean | null;
  driveable: boolean | null;
  operationState: RobotStateEventState | null;
  attemptCancellationReason: AttemptCancellationReason | null;
  undergoingMaintenance: boolean | null;
  timestamp: number;
  status: SupplyStatus;
}
```

### Fleet Service Models (Go)
```go
type FleetRobot struct {
    Serial string
    Healthy bool
    OperationalState foundation.RobotOperationState
    LocationType LocationType
    Latitude float64
    Longitude float64
    ErrHorz float64
    Velocity float64
    HasFood *bool
    NeedsMaintenance *bool
}
```

## Key Concerns Identified

### 1. Multiple Sources of Truth
- **State Service**: Stores state transitions in `stateHistory`
- **Operations Service**: Stores business state in `RobotStateHistory` + ephemeral cache
- **Dispatch Engine**: Stores robot state in `Resource` table (JSON) + `Robot` table
- **Fleet Service**: Stores robot state in DynamoDB
- **MDS Service**: Currently queries Operations service for status

### 2. Data Freshness Issues
- **State Service**: Processes events in real-time but stores in DB
- **Operations Service**: Uses Redis cache with TTL (ephemeral data)
- **Dispatch Engine**: Updates on event receipt, but may have stale data if events are missed
- **Fleet Service**: Processes Kafka messages (may have different latency than RabbitMQ)

### 3. Schema Inconsistencies
- **Operation State**: Different enums across services:
  - State Service: `'GROUNDED' | 'ON_TRIP' | 'PARKED' | 'OFF_DUTY'`
  - Operations: `RobotStateEventState` enum (ON_TRIP, PARKED, GROUNDED, OFF_DUTY, DEPLOYED)
  - Fleet: `foundation.RobotOperationState` (Go enum)
- **Location**: Different structures:
  - State: `{ latitude, longitude, heading, errHorz, updatedAt }`
  - Operations: `{ lat, lng, timestamp }`
  - Dispatch: `Location` type with `geo: { latitude, longitude }`
- **Battery**: Different representations:
  - State: `{ soc, isCharging, sn? }`
  - Operations: `{ currentBatteryPercentage, timestamp }`
  - Dispatch: `number | null` (just SOC)

### 4. Event Processing Differences
- **State Service**: Processes all events, aggregates health, publishes heartbeats
- **Operations Service**: Processes heartbeats and state changes separately
- **Dispatch Engine**: Only processes heartbeats and state changes, ignores other events
- **Trip Monitor**: Only processes heartbeats for trip tracking

### 5. Missing or Incomplete Data
- Some services may not receive all events (e.g., if RabbitMQ consumer fails)
- Different services may process events at different times
- Cache TTLs may cause stale data
- Some services query different sources (e.g., Operations queries State service via RPC)

## Current API Endpoints

### Operations Service
- `GET /robots/:serial/state` - Returns `{ robotState }`
- `GET /robots/:serial/status` - Returns robot status
- `GET /internal/robots` - Returns list of robots with full state
- `GET /robots/:serial/location` - Returns location data

### State Service
- `GET /state/:serial/current` - Returns current state from stateHistory
- `GET /devices/state/latest` - Returns latest state for multiple devices
- RPC endpoints for internal service communication

### Device Service
- `GET /devices/:serial` - Device details
- `GET /devices` - List devices

### MDS Service
- `GET /v1/vehicles` - Vehicle inventory (currently queries Operations)
- `GET /v1/vehicles/status` - Vehicle status (NOT IMPLEMENTED - returns 501)
- `GET /v1/vehicles/:device_id` - Single vehicle
- `GET /v1/trips` - Trip data (from Redshift)

## MDS Requirements (from spec)

### Vehicle States
- `removed`, `available`, `non_operational`, `reserved`, `on_trip`, `stopped`, `non_contactable`, `missing`, `elsewhere`

### Event Types
- `comms_lost`, `comms_restored`, `trip_start`, `trip_end`, `trip_pause`, `trip_resume`, `order_pick_up`, `order_drop_off`, `reservation_start`, `reservation_stop`, `customer_cancellation`, `provider_cancellation`, `driver_cancellation`, `maintenance`, `maintenance_pick_up`, `maintenance_end`, `decommissioned`, `recommission`, `service_start`, `service_end`, `trip_enter_jurisdiction`, `trip_leave_jurisdiction`, `located`, `not_located`, `compliance_pick_up`

### Required Fields
- `device_id`, `provider_id`, `vehicle_state`, `last_event` (with `event_id`, `vehicle_state`, `timestamp`)

## Usage Patterns

### What Services Query Device State?

1. **Dispatch Engine**: 
   - Queries robot supply for planning
   - Needs: location, battery, healthy, operationState, needsMovement, hasFood, tripType
   - Frequency: Real-time during planning cycles

2. **Operations Service**:
   - Queries robot state for operational decisions
   - Needs: All state fields, connectivity, health, location, battery
   - Frequency: On-demand via API, cached for performance

3. **Trip Monitor**:
   - Queries robot state for trip tracking
   - Needs: location, operationState
   - Frequency: Real-time from heartbeats

4. **MDS Service**:
   - Queries robot state for city reporting
   - Needs: vehicle_state, last_event, device_id, location
   - Frequency: On-demand API requests from cities

5. **Fleet Service**:
   - Queries robot state for fleet management
   - Needs: OperationalState, Healthy, Location, HasFood, NeedsMaintenance
   - Frequency: Real-time from Kafka + RabbitMQ

6. **Web Frontend (Mission Control)**:
   - Queries robot state for UI display
   - Needs: All state fields
   - Frequency: Polling/real-time updates

## Recommendations for Device API Schema

Based on usage patterns, a unified Device API should support:

### Core Device State
- `device_id` (serial)
- `location` - Standardized location object
- `battery` - Standardized battery object
- `healthy` - Boolean health status
- `operation_state` - Unified operation state enum
- `connectivity` - Connection status

### Business State
- `needs_movement` - Boolean
- `has_food` - Boolean
- `trip_type` - Enum
- `needs_maintenance` - Boolean
- `needs_pickup` - Boolean
- `driveable` - Boolean
- `undergoing_maintenance` - Boolean

### MDS-Specific Fields
- `vehicle_state` - MDS vehicle state enum
- `last_event` - Most recent event with timestamp
- `provider_id` - Provider identifier

### Metadata
- `last_updated` - Timestamp of last update
- `data_freshness` - Age of data
- `source` - Where the data came from

## Deep Dive Findings

### 1. State Determination Logic

#### Operations Service State Determination
The Operations service determines robot state through a combination of sources:

**Operational Readiness Check** (`getRobotOperationalReadinessIssue`):
1. **Connectivity Check**: Queries `robotEphemeralDataService.isConnected(serial)` - checks Redis set
2. **Trip Status**: Checks if robot is on trip via `robotEphemeralDataService.robotIsOnTrip(serial)`
3. **Ephemeral State**: Gets `robotEphemeralState` from cache (TTL: `ROBOT_EPHEMERAL_STATE_TTL_IN_SECONDS`)
   - Falls back to querying State Service via RPC if cache miss
   - Checks `robotState` field (GROUNDED, ON_TRIP, PARKED, etc.)
4. **Health Status**: Gets `ephemeralHealthInfo` from cache
5. **State History**: Queries `RobotStateHistory` table for latest record
   - Checks `undergoingMaintenance` and `needsMaintenance` flags

**State Sources Priority**:
- Ephemeral cache (Redis) - fastest, may be stale
- State Service RPC - fallback if cache miss
- State History DB - for business logic flags (maintenance, etc.)

**Key State Fields**:
- `operationState`: ON_TRIP | PARKED | GROUNDED | OFF_DUTY | DEPLOYED
- `robotState`: GROUNDED | ON_TRIP | PARKED | etc. (from State Service)
- `needsMaintenance`: boolean
- `undergoingMaintenance`: boolean
- `hasFood`: boolean
- `needsMovement`: boolean
- `driveable`: boolean

#### State Service State Machine
The State Service maintains a state machine with these transitions:
- **States**: OFF_DUTY, PARKED, ON_TRIP, GROUNDED
- **Transitions**: 
  - HARD_RESET → OFF_DUTY (from any state)
  - DEPLOY_FO_APP → PARKED (from OFF_DUTY)
  - UNDEPLOY_FO_APP → OFF_DUTY (from PARKED)
  - TRIP_ASSIGNED → ON_TRIP (from OFF_DUTY or PARKED)
  - TRIP_COMPLETED → PARKED (from ON_TRIP)
  - CRITICAL_ERROR_DETECTED → GROUNDED (from ON_TRIP, PARKED, or OFF_DUTY)
  - RECOVERING → OFF_DUTY (from GROUNDED)

### 2. Dispatch Engine Usage Patterns

#### Robot Filtering for Planning
Dispatch Engine uses `SupplyService.getRobotUnusableReasons()` to filter robots:

**Usability Checks** (varies by demand type):
- **Always Required**:
  - `Online`: Robot must have synced within last 1 minute (`lastSynced` check)
  - `HasLocation`: Robot must have a location
  - `Deployed`: Robot must have a deployment location
- **For Pickup/Delivery**:
  - `Healthy`: Robot must be healthy
  - `DeployedToParkingLot`: Robot must be deployed to a parking lot
  - `NotNeedsMaintenance`: Robot must not need maintenance
- **For Deployment**:
  - `Healthy`: Robot must be healthy

#### Planning Logic
The planner filters robots by:
1. **Distance**: Filters robots within `maxSearchDistanceMiles` of origin
2. **Usability**: Applies `getRobotUnusableReasons()` checks
3. **Scheduled Demands**: 
   - Rejects robots with scheduled delivery
   - Rejects robots with scheduled pickup (unless P2P enabled)
   - Considers scheduled movement (may act as pickup if destination matches)
4. **Active Demands**: 
   - Uses active demand completion time as robot position time
   - May use active demand as existing pickup if destination matches origin

**Critical Robot Fields for Planning**:
- `location`: Current location (used for distance calculations)
- `deployed`: Deployment location (used for filtering and position tracking)
- `healthy`: Health status (required for delivery/pickup)
- `needsMaintenance`: Maintenance flag (blocks delivery/pickup)
- `lastSynced`: Last sync time (must be < 1 minute for "online")
- `activeDemand`: Current active demand (affects position/time)
- `scheduledDemands`: Future demands (pickup, delivery, movement)
- `operationState`: Operational state (affects usability)
- `battery`: Battery level (stored but not actively filtered)

**Data Freshness Requirements**:
- `lastSynced` must be within 1 minute for robot to be considered "online"
- Location data should be recent (heartbeats come every ~10 seconds)
- State changes should propagate quickly for planning accuracy

### 3. MDS Mapping Requirements

#### Current MDS Implementation
The MDS service currently:
- Queries Operations service `/internal/robots` endpoint
- Maps `RobotState.OperationState` to MDS `vehicle_state`
- Uses `GetDeviceIDForVehicle()` to generate stable device IDs

#### State Mapping (`normalizeVehicleState` function):
```go
// Internal State → MDS Vehicle State
"PARKED" or "AVAILABLE" → "available"
"EN_ROUTE" or "TRIP" or "IN_DELIVERY" → "trip"
"MAINTENANCE" or "UNDER_MAINTENANCE" → "maintenance"
// All others → lowercase of input
```

#### Required MDS Fields
- `device_id`: Stable device identifier (from `GetDeviceIDForVehicle`)
- `provider_id`: Provider identifier
- `vehicle_state`: One of: `removed`, `available`, `non_operational`, `reserved`, `on_trip`, `stopped`, `non_contactable`, `missing`, `elsewhere`
- `last_event`: Object with:
  - `event_id`: Unique event identifier
  - `vehicle_state`: Current vehicle state
  - `timestamp`: Event timestamp (milliseconds)

#### MDS Event Types Needed
Based on the MDS spec, the following event types are relevant:
- `trip_start`, `trip_end`, `trip_pause`, `trip_resume`
- `order_pick_up`, `order_drop_off`
- `reservation_start`, `reservation_stop`
- `customer_cancellation`, `provider_cancellation`
- `maintenance`, `maintenance_pick_up`, `maintenance_end`
- `service_start`, `service_end`
- `comms_lost`, `comms_restored`
- `trip_enter_jurisdiction`, `trip_leave_jurisdiction`

**Current Gap**: MDS `/v1/vehicles/status` endpoint is NOT IMPLEMENTED (returns 501)

### 4. Event Processing Timing

#### Event Flow Timeline
1. **Robot sends heartbeat** → AWS IoT (MQTT)
2. **IoT Bridge** → SQS queue (`stateQueueUrl`)
3. **State Service (`IotStreamerService`)** processes message:
   - Parses heartbeat data
   - Aggregates health status (checks DriveU status from DynamoDB)
   - Detects battery swaps
   - Saves to `heartbeat` table via `HeartbeatService`
   - Publishes to RabbitMQ `IoT.Heartbeat` exchange
4. **Multiple consumers** receive heartbeat:
   - **Operations Service**: Updates ephemeral cache, saves heartbeat to DB
   - **Dispatch Engine**: Updates robot supply in `Resource` table
   - **Trip Monitor**: Updates trip state
   - **Fleet Service**: Updates DynamoDB (separate Kafka consumer)

#### Timing Characteristics
- **Heartbeat frequency**: ~10 seconds
- **SQS processing**: Batch size 10, timeout 30s
- **RabbitMQ**: Near-instant propagation
- **Cache TTLs**: 
  - Ephemeral state: `ROBOT_EPHEMERAL_STATE_TTL_IN_SECONDS` (unknown, likely 60-300s)
  - Ephemeral health: Similar TTL
- **Database writes**: Synchronous, may have replication lag

#### Potential Race Conditions
1. **State updates vs. heartbeats**: State changes may arrive before/after heartbeats
2. **Multiple consumers**: Different services may see different states if processing at different times
3. **Cache invalidation**: Cache may be stale if events are missed
4. **Database replication**: Read replicas may lag behind primary

### 5. API Query Patterns

#### Current Usage Patterns

**Single Device Queries**:
- Operations: `GET /robots/:serial/state`, `GET /robots/:serial/status`
- State: `GET /state/:serial/current`
- Device: `GET /devices/:serial`
- MDS: `GET /v1/vehicles/:device_id`

**Bulk Queries**:
- Operations: `GET /internal/robots` (returns all robots)
- Dispatch Engine: `getAll(ResourceType.Robot)` - queries all robots for planning
- State: `GET /devices/state/latest` (with serials param)
- MDS: `GET /v1/vehicles` (returns all vehicles for city)

**Filtered Queries**:
- Operations: `findBots(criteria)` - filters by:
  - `isOnTrip`: boolean
  - `isOnTask`: boolean
  - `isHealthy`: boolean
  - `isConnectivityHealthy`: boolean
  - `isGrounded`: boolean
  - `range`: geographic radius
- Dispatch Engine: Filters robots by distance, usability checks, scheduled demands

**Real-time Requirements**:
- Dispatch Engine: Needs fresh data for planning (checks `lastSynced < 1 minute`)
- Operations: Uses cache for performance but needs accurate state
- Mission Control UI: Polls for updates, needs recent data
- MDS: External API, needs rate limiting/DDoS protection

#### Data Freshness Expectations
- **Dispatch Engine**: `lastSynced` must be < 1 minute
- **Operations**: Cache TTL likely 60-300 seconds, but queries State Service RPC on cache miss
- **State Service**: Real-time processing, stores in DB immediately
- **Fleet Service**: Processes Kafka messages (may have different latency)

## Recommendations for Device API Schema

Based on usage patterns, a unified Device API should support:

### Core Device State
- `device_id` (serial)
- `location` - Standardized location object with timestamp
- `battery` - Standardized battery object with timestamp
- `healthy` - Boolean health status
- `operation_state` - Unified operation state enum
- `connectivity` - Connection status (online/offline)
- `last_synced` - Timestamp of last update

### Business State
- `needs_movement` - Boolean
- `has_food` - Boolean
- `trip_type` - Enum (DELIVERY, RETURN, DEPLOYMENT, PICKUP, NONE)
- `needs_maintenance` - Boolean
- `needs_pickup` - Boolean
- `driveable` - Boolean
- `undergoing_maintenance` - Boolean
- `deployed_location` - Deployment location info

### MDS-Specific Fields
- `vehicle_state` - MDS vehicle state enum
- `last_event` - Most recent event with timestamp and event_id
- `provider_id` - Provider identifier

### Metadata
- `last_updated` - Timestamp of last update
- `data_freshness` - Age of data (for monitoring)
- `source` - Where the data came from (for debugging)

### API Endpoints Needed

**Single Device**:
- `GET /devices/:device_id` - Full device state
- `GET /devices/:device_id/state` - Current state only
- `GET /devices/:device_id/status` - Status summary (for MDS compatibility)

**Bulk Queries**:
- `GET /devices` - List all devices (with filtering)
- `POST /devices/batch` - Get multiple devices by IDs
- `GET /devices/available` - Get available robots (for Dispatch Engine)

**Filtering Support**:
- Query params: `healthy`, `operation_state`, `deployed_location`, `geographic_bounds`, `last_synced_after`
- Support pagination for large result sets

**Real-time Updates**:
- WebSocket support for Mission Control UI
- Webhook support for state changes
- Rate limiting for external consumers (MDS)

## Next Steps

1. **Schema design**:
   - Design unified device state schema
   - Map existing schemas to unified schema
   - Design API request/response formats

2. **Architecture design**:
   - Determine where Device API service should live
   - Design data aggregation strategy (single source of truth)
   - Design caching strategy (maintain freshness requirements)
   - Design event processing strategy (ensure consistency)

3. **Migration plan**:
   - Identify all current usages
   - Plan migration path for each service
   - Design backward compatibility layer

