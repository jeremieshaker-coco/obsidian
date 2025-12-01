# Device API: Technical Design

## Overview

The Device API is a new gRPC service that serves as the single source of truth for robot/device state. It aggregates state from the State Service and exposes a unified API for all consumers.

## API Design

### Language/Technology

| Option | Pros | Cons |
|--------|------|------|
| **Go** | Better performance, lower latency, native gRPC support, aligns with Fleet/MDS services, lower memory footprint | Different language from main consumers (Dispatch Engine, Operations), less code sharing |
| **TypeScript (NestJS)** | Same language as Dispatch Engine/Operations, code sharing, team familiarity, consistent patterns | Higher latency, more memory, gRPC requires additional setup |

**Decision**: **Go**

**Rationale**: Device API is a read-heavy, high-throughput query service where latency matters. Go's performance characteristics (lower latency, better concurrency, smaller memory footprint) make it the better fit. The gRPC interface provides clean separation, so TypeScript consumers can easily integrate via generated clients. This also aligns with Fleet Service and other core infrastructure services.

### Protocol

The API is exposed via **gRPC** as the primary interface for internal services. gRPC provides:
- Better performance (binary protocol, HTTP/2)
- Type safety (via Protocol Buffers)
- Streaming capabilities for real-time updates
- Strong backward compatibility guarantees

### AIP Compliance

This API follows [Google's API Improvement Proposals (AIPs)](https://google.aip.dev/) for consistency and best practices:

- **AIP-122**: Resource-oriented design with hierarchical resource names (`devices/{device_id}`)
- **AIP-131**: Standard methods (`Get`, `List`, `BatchGet`)
- **AIP-132**: Request/response naming conventions (`GetDeviceRequest`, `ListDevicesResponse`)
- **AIP-134**: Standard fields (`create_time`, `update_time`)
- **AIP-158**: Token-based pagination (`page_token`, `page_size`)
- **AIP-161**: Field masks for partial responses
- **AIP-190**: Naming conventions (American English, clear and consistent)
- **AIP-231**: Batch methods return resources in requested order

### gRPC Service Definition

This API follows [Google's API Improvement Proposals (AIPs)](https://google.aip.dev/) for consistency and best practices.

```protobuf
syntax = "proto3";

package deviceapi.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";

service DeviceService {
  // Standard method: Get a single device (AIP-131)
  rpc GetDevice(GetDeviceRequest) returns (Device);

  // Custom method: Get device state summary (lightweight sub-resource)
  rpc GetDeviceState(GetDeviceStateRequest) returns (DeviceState);

  // Standard method: List devices with filtering (AIP-131)
  rpc ListDevices(ListDevicesRequest) returns (ListDevicesResponse);

  // Standard method: Batch get multiple devices (AIP-231)
  rpc BatchGetDevices(BatchGetDevicesRequest) returns (BatchGetDevicesResponse);

  // Custom method: Get available robots (for Dispatch Engine)
  rpc GetAvailableDevices(GetAvailableDevicesRequest) returns (GetAvailableDevicesResponse);

  // Custom method: Stream device updates (real-time)
  rpc StreamDeviceUpdates(StreamDeviceUpdatesRequest) returns (stream DeviceUpdate);
}

// Request/Response Messages (AIP-132: Request naming conventions)

message GetDeviceRequest {
  // Resource name: devices/{device_id} (AIP-122)
  string name = 1; // Format: "devices/{device_id}" where device_id is robot serial number
  
  // Field mask for partial responses (AIP-161)
  google.protobuf.FieldMask read_mask = 2;
}

message GetDeviceStateRequest {
  string name = 1; // Format: "devices/{device_id}"
  google.protobuf.FieldMask read_mask = 2;
}

message ListDevicesRequest {
  // Parent resource (optional, for future hierarchical support)
  string parent = 1; // Format: "locations/{location_id}" or empty for all devices
  
  // Filters
  optional bool healthy = 2;
  optional OperationState operation_state = 3;
  optional string deployed_location_id = 4;
  optional GeographicBounds geographic_bounds = 5;
  optional google.protobuf.Timestamp last_synced_after = 6;
  optional bool has_food = 7;
  optional bool needs_maintenance = 8;
  
  // Pagination (AIP-158: Token-based pagination)
  int32 page_size = 10; // Default: 50, Max: 1000
  string page_token = 11; // Token from previous response
  
  // Field mask for partial responses (AIP-161)
  google.protobuf.FieldMask read_mask = 12;
}

message ListDevicesResponse {
  repeated Device devices = 1;
  string next_page_token = 2; // Token for next page (AIP-158)
}

message BatchGetDevicesRequest {
  // Resource names: devices/{device_id} (AIP-231)
  repeated string names = 1; // Format: ["devices/{device_id}", ...]
  
  // Field mask for partial responses (AIP-161)
  google.protobuf.FieldMask read_mask = 2;
}

message BatchGetDevicesResponse {
  // Devices returned in same order as requested (AIP-231)
  // Missing devices are omitted (not included in separate list)
  repeated Device devices = 1;
}

message GetAvailableDevicesRequest {
  string parent = 1; // Optional: "locations/{location_id}"
  optional string deployed_location_id = 2;
  optional GeographicBounds geographic_bounds = 3;
  
  // Pagination
  int32 page_size = 4; // Default: 50, Max: 1000
  string page_token = 5;
  
  google.protobuf.FieldMask read_mask = 6;
}

message GetAvailableDevicesResponse {
  repeated Device devices = 1;
  string next_page_token = 2;
}

message StreamDeviceUpdatesRequest {
  // Resource names: devices/{device_id} (empty = all devices)
  repeated string names = 1; // Format: ["devices/{device_id}", ...]
  
  optional google.protobuf.Timestamp since = 2; // Only updates after this time
}

message DeviceUpdate {
  Device device = 1;
  google.protobuf.Timestamp update_time = 2; // AIP-134: Use update_time instead of updated_at
}

// Core Data Types

message Device {
  // Resource name (AIP-122): devices/{device_id}
  string name = 1; // Format: "devices/{device_id}"
  
  // Core Identity
  string device_id = 2; // Serial number (deprecated, use name instead)
  string provider_id = 3; // For MDS compatibility
  
  // Robot Telemetry (from State Service)
  Location location = 4;
  Battery battery = 5;
  bool healthy = 6;
  OperationState operation_state = 7;
  Connectivity connectivity = 8;
  optional DriveUStatus driveu_status = 9; // DriveU remote control status
  
  // Trip Context (from Operations Service)
  optional bool needs_movement = 10;
  optional bool has_food = 11;
  TripType trip_type = 12;
  optional bool needs_maintenance = 13;
  optional bool needs_pickup = 14;
  optional bool driveable = 15;
  optional bool undergoing_maintenance = 16;
  
  // Fleet Assignment (from Fleet Service)
  Deployment deployed_location = 17;
  
  // MDS Fields
  optional string vehicle_state = 18; // MDS vehicle_state
  optional Event last_event = 19; // MDS event
  
  // Metadata (AIP-134: Standard fields)
  google.protobuf.Timestamp create_time = 20; // When device was first seen
  google.protobuf.Timestamp update_time = 21; // Last update time
  google.protobuf.Timestamp last_synced = 22; // Last sync with robot
  int32 data_freshness_seconds = 23; // Age of data in seconds
}

message DeviceState {
  string name = 1; // Format: "devices/{device_id}"
  string device_id = 2; // Serial number (deprecated, use name instead)
  OperationState operation_state = 3;
  Connectivity connectivity = 4;
  bool healthy = 5;
  google.protobuf.Timestamp last_synced = 6;
  google.protobuf.Timestamp update_time = 7;
}

message Location {
  double latitude = 1;
  double longitude = 2;
  optional double heading = 3;
  optional double err_horz = 4; // Horizontal error in meters
  google.protobuf.Timestamp updated_at = 5;
}

message Battery {
  int32 soc = 1; // State of charge (0-100)
  bool is_charging = 2;
  optional string serial_number = 3; // Battery serial
  google.protobuf.Timestamp updated_at = 4;
}

message Connectivity {
  bool online = 1;
  google.protobuf.Timestamp last_seen = 2;
}

message DriveUStatus {
  string streamer_status = 1; // offline, connectedToNode, standby, etc.
  google.protobuf.Timestamp updated_at = 2;
  optional string previous_status = 3;
}

message Deployment {
  string id = 1;
  LocationType type = 2;
  optional string name = 3;
}

message Event {
  string event_id = 1;
  string vehicle_state = 2; // MDS vehicle_state
  int64 timestamp = 3; // Unix timestamp in milliseconds
}

message Telemetry {
  string telemetry_id = 1;
  int64 timestamp = 2; // Unix timestamp in milliseconds
}

message GeographicBounds {
  double min_latitude = 1;
  double max_latitude = 2;
  double min_longitude = 3;
  double max_longitude = 4;
}

// Enums

enum OperationState {
  OPERATION_STATE_UNSPECIFIED = 0;
  OPERATION_STATE_ON_TRIP = 1;
  OPERATION_STATE_PARKED = 2;
  OPERATION_STATE_GROUNDED = 3;
  OPERATION_STATE_OFF_DUTY = 4;
  OPERATION_STATE_DEPLOYED = 5;
}

enum TripType {
  TRIP_TYPE_UNSPECIFIED = 0;
  TRIP_TYPE_DELIVERY = 1;
  TRIP_TYPE_RETURN = 2;
  TRIP_TYPE_DEPLOYMENT = 3;
  TRIP_TYPE_PICKUP = 4;
  TRIP_TYPE_NONE = 5;
}

enum LocationType {
  LOCATION_TYPE_UNSPECIFIED = 0;
  LOCATION_TYPE_PARKING_LOT = 1;
  LOCATION_TYPE_MRO = 2;
  LOCATION_TYPE_POD = 3;
}
```

### Error Handling (AIP-193)

The API follows standard gRPC error codes and Google's error model:

- **NOT_FOUND (5)**: Device not found (invalid `device_id` in resource name)
- **INVALID_ARGUMENT (3)**: Invalid request parameters (e.g., invalid filter values, invalid page_token)
- **RESOURCE_EXHAUSTED (8)**: Rate limit exceeded
- **UNAVAILABLE (14)**: Service temporarily unavailable
- **INTERNAL (13)**: Internal server error

Error responses include:
- `code`: gRPC status code
- `message`: Human-readable error message
- `details`: Additional error details (optional)

Example error response:
```json
{
  "error": {
    "code": 404,
    "message": "Device not found: devices/INVALID_SERIAL",
    "status": "NOT_FOUND"
  }
}
```

### Field Masks (AIP-161)

Field masks allow clients to request partial responses, reducing payload size and improving performance:

```protobuf
// Request only location and battery fields
GetDeviceRequest {
  name: "C10190"
  read_mask: {
    paths: ["location", "battery"]
  }
}
```

Supported in:
- `GetDevice` - Request specific fields
- `ListDevices` - Request specific fields for all devices
- `BatchGetDevices` - Request specific fields for batch requests

### REST Endpoints (via gRPC-Gateway, Optional)

For services that prefer REST over gRPC, REST endpoints can be exposed via gRPC-Gateway (AIP-127: HTTP and gRPC Transcoding):

- `GET /v1/devices/{device_id}` - Full device state
- `GET /v1/devices/{device_id}?read_mask=location,battery` - Partial response via field mask
- `GET /v1/devices/{device_id}/state` - Device state summary
- `GET /v1/devices?page_size=50&page_token=...` - List devices (with filtering, pagination via `page_token`)
- `POST /v1/devices:batchGet` - Batch get devices
- `GET /v1/devices:available` - Get available robots

**Note**: Resource names use the format `devices/{device_id}` per AIP-122. The REST API maps these to gRPC resource names automatically via gRPC-Gateway.

## Internal Architecture

### State Persistence Strategy

**Decision: Device API does NOT maintain its own database. It queries State Service database directly.**

#### Rationale

1. **Single Source of Truth Requirement**: The PRD requires a single source of truth. State Service database (`StateHistory` table) is the authoritative source for robot state. Maintaining a separate database would create data duplication and potential inconsistencies.

2. **Data Freshness Requirement**: The requirement states "no regression in data freshness." Querying State Service directly ensures we always get the latest state, with caching only used for performance optimization, not as a source of truth.

3. **Simplicity**: Avoiding a separate database reduces:
   - Data synchronization complexity
   - Potential for inconsistencies
   - Storage costs
   - Maintenance overhead

4. **State Aggregation**: Device API aggregates state from multiple sources:
   - **State Service DB**: Robot telemetry (location, battery, operation state, connectivity)
   - **Operations Service DB**: Trip context (`needsMovement`, `hasFood`, `tripType`, etc.)
   - **Fleet Service**: Fleet assignment (deployment location)
   
   Maintaining separate copies of all this data would be complex and error-prone.

#### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Device API Service                                          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  In-Memory Cache (5s TTL)                            │  │
│  │  • Key: device_id                                    │  │
│  │  • Value: Aggregated Device state                    │  │
│  │  • Invalidation: On RabbitMQ event receipt           │  │
│  └──────────────────────────────────────────────────────┘  │
│                        │                                    │
│                        ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  State Aggregation Layer                              │  │
│  │  • Queries State Service DB (read replica)            │  │
│  │  • Queries Operations Service DB                     │  │
│  │  • Queries Fleet Service                             │  │
│  │  • Aggregates into unified Device model              │  │
│  └──────────────────────────────────────────────────────┘  │
│                        │                                    │
│                        ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  RabbitMQ Event Listener                              │  │
│  │  • Subscribes to: IoT.Heartbeat                      │  │
│  │  • Subscribes to: IoT.HealthStateChanged             │  │
│  │  • Subscribes to: Robots.StateChange                 │  │
│  │  • Invalidates cache on event receipt                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ State        │ │ Operations   │ │ Fleet       │
│ Service DB   │ │ Service DB   │ │ Service     │
│ (Read        │ │              │ │             │
│  Replica)    │ │              │ │             │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Caching Strategy

#### In-Memory Cache

**Purpose**: Reduce database load and improve response times for frequently accessed devices.

**Technology**: **Redis**

Redis is the recommended caching solution because:
- Shared cache across Device API instances (supports horizontal scaling)
- Consistent with existing infrastructure (used by Operations Service, Fleet Service)
- Provides atomic operations and TTL support
- Well-supported in Go ecosystem (`go-redis/redis`)

**Implementation**:
- **Key Format**: `device:{device_id}` (e.g., `device:C10190`)
- **TTL**: 5 seconds (less than heartbeat frequency of 10 seconds)
- **Redis Operations**: `SETEX` for TTL and `GET` for retrieval

**Cache Invalidation**:
1. **Time-based**: Automatic expiration after 5 seconds (via Redis TTL)
2. **Event-based**: On RabbitMQ event receipt for that device
3. **Manual**: On cache miss, fetch fresh data and populate cache

**Implementation Pattern**:
```go
func (s *DeviceService) GetDevice(ctx context.Context, deviceID string) (*pb.Device, error) {
    cacheKey := fmt.Sprintf("device:%s", deviceID)
    
    // Try cache first
    cached, err := s.redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var device pb.Device
        if err := proto.Unmarshal([]byte(cached), &device); err == nil {
            return &device, nil
        }
    }
    
    // Cache miss - fetch and aggregate
    device, err := s.aggregateDeviceState(ctx, deviceID)
    if err != nil {
        return nil, err
    }
    
    // Cache with TTL
    data, _ := proto.Marshal(device)
    s.redis.SetEx(ctx, cacheKey, data, 5*time.Second)
    
    return device, nil
}

// Invalidate on RabbitMQ event
func (s *DeviceService) handleDeviceEvent(deviceID string) {
    cacheKey := fmt.Sprintf("device:%s", deviceID)
    s.redis.Del(context.Background(), cacheKey)
}
```

**Why 5 seconds?**
- Heartbeats arrive every ~10 seconds
- 5-second TTL ensures cache is refreshed before data becomes stale
- Balances freshness with performance (reduces DB queries by ~80% for frequently accessed devices)

#### Cache Miss Handling

When cache miss occurs:
1. Query State Service database (read replica) for robot telemetry
2. Query Operations Service database for trip context
3. Query Fleet Service for fleet assignment
4. Aggregate into unified `Device` model
5. Store in cache with 5-second TTL
6. Return to client

**Performance Optimization**: For `BatchGetDevices` and `ListDevices`, we:
- Check cache for all requested devices
- Batch query database only for cache misses
- Merge cached and fresh data
- Update cache with fresh data

### Database Query Strategy

#### Read Replicas

**Decision**: Use read replicas of State Service database for Device API queries.

**Rationale**:
- Reduces load on primary State Service database
- Improves query performance (read replicas can be optimized for reads)
- Maintains single source of truth (replicas are eventually consistent)
- Allows horizontal scaling of read capacity

#### Query Patterns

1. **Single Device Query** (`GetDevice`):
   ```sql
   -- Query State Service DB (read replica)
   SELECT * FROM "StateHistory" 
   WHERE serial = $1 
   ORDER BY date DESC 
   LIMIT 1;
   
   -- Query Operations Service DB
   SELECT * FROM "RobotStateHistory" 
   WHERE "robotSerial" = $1 
   ORDER BY "createdAt" DESC 
   LIMIT 1;
   
   -- Query Fleet Service (via API)
   GET /fleet/robots/{device_id}/deployment
   ```

2. **Bulk Query** (`BatchGetDevices`):
   - Use `IN` clause for efficient batch queries
   - Parallel queries to different services
   - Merge results client-side

3. **List Query** (`ListDevices`):
   - Apply filters at database level (e.g., `WHERE healthy = true`)
   - Use pagination with `LIMIT` and `OFFSET` (or cursor-based)
   - Cache frequently accessed filter combinations

### State Aggregation Logic

The Device API aggregates state from multiple sources into a unified `Device` model:

#### 1. Robot Telemetry (from State Service DB)

**Fields**:
- `operation_state`: From `StateHistory.state`
- `location`: From latest heartbeat (stored in DynamoDB or State Service)
- `battery`: From latest heartbeat
- `connectivity`: Calculated from last heartbeat timestamp
- `driveu_status`: From DriveU events (streamer status: offline, connectedToNode, standby, etc.)

**DriveU Status Integration**:
- DriveU is a video streaming/remote control system that allows pilots to remotely control robots
- DriveU status events come through RabbitMQ (`ConnectivityRawStatusChanged` exchange)
- State Service processes these events and stores DriveU status in DynamoDB
- Device API queries State Service for latest DriveU status
- DriveU status affects robot health determination (offline >90s = CRITICAL_FAULT)

#### 2. Trip Context (from Operations Service DB)

**Fields**:
- `needsMovement`, `hasFood`, `tripType`: From `RobotStateHistory`
- `needsMaintenance`, `needsPickup`, `driveable`: From `RobotStateHistory`
- `undergoingMaintenance`: From `RobotStateHistory`

#### 3. Fleet Assignment (from Fleet Service)

**Fields**:
- `deployed_location`: Current deployment location (parking lot, MRO, POD)

#### 4. Aggregation

```go
func (s *DeviceService) aggregateDeviceState(ctx context.Context, deviceID string) (*pb.Device, error) {
    // Parallel queries to all data sources
    var (
        telemetry      *RobotTelemetry
        tripContext    *TripContext
        fleetAssignment *FleetAssignment
    )
    
    g, ctx := errgroup.WithContext(ctx)
    
    g.Go(func() error {
        var err error
        telemetry, err = s.stateRepo.GetLatestTelemetry(ctx, deviceID)
        return err
    })
    
    g.Go(func() error {
        var err error
        tripContext, err = s.operationsRepo.GetTripContext(ctx, deviceID)
        return err
    })
    
    g.Go(func() error {
        var err error
        fleetAssignment, err = s.fleetClient.GetRobotDeployment(ctx, deviceID)
        return err
    })
    
    if err := g.Wait(); err != nil {
        return nil, err
    }
    
    // Build unified Device model
    return &pb.Device{
        Name:           fmt.Sprintf("devices/%s", deviceID),
        DeviceId:       deviceID,
        OperationState: mapToOperationState(telemetry.State),
        Location:       transformLocation(telemetry.Location),
        Battery:        transformBattery(telemetry.Battery),
        Connectivity:   calculateConnectivity(telemetry.LastHeartbeat),
        Healthy:        telemetry.Healthy,
        DriveUStatus:   transformDriveUStatus(telemetry.DriveU),
        NeedsMovement:  tripContext.NeedsMovement,
        HasFood:        tripContext.HasFood,
        TripType:       mapToTripType(tripContext.TripType),
        // ... other fields
        DeployedLocation: transformDeployment(fleetAssignment),
        UpdateTime:       timestamppb.New(telemetry.UpdatedAt),
        LastSynced:       timestamppb.New(telemetry.LastHeartbeat),
        DataFreshnessSeconds: int32(time.Since(telemetry.LastHeartbeat).Seconds()),
    }, nil
}
```

### Performance Considerations

#### Query Optimization

1. **Database Indexes**: Ensure proper indexes exist:
   - `StateHistory`: `(serial, date DESC)`
   - `RobotStateHistory`: `(robotSerial, createdAt DESC)`

2. **Connection Pooling**: Use connection pooling for database connections

3. **Parallel Queries**: Query multiple services in parallel using `errgroup`:
   ```go
   g, ctx := errgroup.WithContext(ctx)
   g.Go(func() error { telemetry, err = s.stateRepo.Get(ctx, deviceID); return err })
   g.Go(func() error { tripCtx, err = s.opsRepo.Get(ctx, deviceID); return err })
   g.Go(func() error { fleet, err = s.fleetClient.Get(ctx, deviceID); return err })
   if err := g.Wait(); err != nil { return nil, err }
   ```

4. **Bulk Operations**: For `BatchGetDevices`, use batch queries:
   ```sql
   SELECT * FROM "StateHistory" 
   WHERE serial IN ($1, $2, $3, ...)
   AND date IN (
     SELECT MAX(date) FROM "StateHistory" 
     WHERE serial IN ($1, $2, $3, ...)
     GROUP BY serial
   );
   ```

#### Scalability

1. **Horizontal Scaling**: Device API can scale horizontally (stateless service)
2. **Read Replicas**: Add more read replicas as query volume increases
3. **Cache Distribution**: Use distributed cache (Redis) if scaling beyond single instance
4. **Query Limits**: Enforce `page_size` limits (max 1000) to prevent expensive queries

### Failure Handling

#### State Service DB Unavailable

- **Cache Hit**: Return cached data (may be up to 5 seconds stale)
- **Cache Miss**: Return `UNAVAILABLE` error (don't serve stale data beyond TTL)

#### Operations Service DB Unavailable

- **Partial Response**: Return device with robot telemetry only
- **Trip Context Fields**: Set to `null` or last known values from cache
- **Log Warning**: Log the partial response for monitoring

#### RabbitMQ Events Delayed/Missing

- **Cache Still Valid**: Cache TTL ensures freshness even if events are delayed
- **Database Fallback**: Cache miss queries database directly
- **Monitoring**: Alert on RabbitMQ consumer lag

## Data Processing Pipeline

### Overview

```
MQTT/IoT Events (SQS)
    ↓
State Service (IotStreamerService)
    ├─→ Processes events
    ├─→ Updates AWS IoT Device Shadow
    ├─→ Stores state in database (StateHistory table)
    └─→ Publishes to RabbitMQ (backward compatibility)
            ↓
    Device API Service
        ├─→ Subscribes to RabbitMQ events (cache invalidation)
        │   • IoT.Heartbeat
        │   • IoT.HealthStateChanged
        │   • Robots.StateChange
        │   • ConnectivityRawStatusChanged (for DriveU status)
        ├─→ Queries State Service DB (read replica) for robot telemetry
        ├─→ Queries State Service for DriveU status (via StateTrackerService)
        ├─→ Queries Operations Service DB for trip context
        ├─→ Queries Fleet Service for fleet assignment
        ├─→ Aggregates into unified Device model
        ├─→ Maintains in-memory cache (5s TTL)
        └─→ Serves gRPC requests
```

### Event Flow

1. **MQTT/IoT Events** arrive via SQS queues (`eventsQueueUrl`, `stateQueueUrl`)
2. **State Service** (`IotStreamerService`) processes events:
   - Heartbeats: Updates location, battery, connectivity
   - Lid events: Tracks lid state changes
   - Pin entry events: Tracks pin entry events
   - State changes: Updates operation state
3. **State Service** stores authoritative state:
   - Database: `stateHistory` table with state transitions
   - AWS IoT Device Shadow: Updates robot attributes
   - RabbitMQ: Publishes events for backward compatibility
     - `IoT.Heartbeat` exchange
     - `IoT.HealthStateChanged` exchange
     - `Robots.StateChange` exchange
4. **Device API Service** aggregates state:
   - Subscribes to RabbitMQ events for real-time updates
   - Queries State Service database for authoritative state
   - Maintains in-memory cache (5-second TTL)
   - Updates cache on RabbitMQ events
   - Serves gRPC requests from cache or database

### State Aggregation Logic

The Device API aggregates state from multiple sources:

1. **Robot Telemetry** (from State Service):
   - Location: Latest GPS coordinates from heartbeat
   - Battery: Latest battery state from heartbeat
   - Connectivity: Online status based on last heartbeat time
   - Operation State: Latest state transition from `stateHistory`
   - DriveU Status: Remote control streamer status (from DriveU events via RabbitMQ)

2. **Trip Context** (from Operations Service):
   - `needsMovement`, `hasFood`, `tripType`: From `RobotStateHistory`
   - `needsMaintenance`, `needsPickup`, `driveable`: From `RobotStateHistory`
   - `undergoingMaintenance`: From `RobotStateHistory`

3. **Fleet Assignment** (from Fleet Service):
   - Current deployment location (parking lot, MRO, POD)

4. **MDS Mapping**:
   - Maps internal `operationState` to MDS `vehicle_state`:
     - `PARKED` → `available`
     - `ON_TRIP` → `on_trip`
     - `GROUNDED` → `non_operational`
     - `OFF_DUTY` → `non_operational`
     - `DEPLOYED` → `available`

### Cache Strategy

See [Internal Architecture](#internal-architecture) section for detailed caching strategy.

**Summary**:
- **In-memory cache**: 5-second TTL for frequently accessed devices
- **Cache invalidation**: On RabbitMQ event receipt + time-based expiration
- **Cache miss**: Query State Service DB (read replica) + Operations Service DB + Fleet Service
- **Freshness guarantee**: Data freshness < 10 seconds (heartbeat frequency)
- **No separate database**: Device API queries source databases directly to maintain single source of truth

## Consumer Migration Plans

### 1. Dispatch Engine

#### Current Usage

Dispatch Engine currently:
- Queries `Robot` table from its own database
- Processes `Robots.StateChange` RabbitMQ events to update robot state
- Uses `SupplyService.robots()` to get available robots
- Filters robots by usability checks (online, location, deployed, healthy, etc.)

#### Migration Steps

1. **Add Device API Client**:
   - Add gRPC client dependency
   - Create `DeviceApiClient` service wrapper

2. **Replace Robot Queries**:
   - Replace `SupplyService.robots()` with `DeviceApiClient.GetAvailableDevices()`
   - Replace `SupplyService.robot(serial)` with `DeviceApiClient.GetDevice(serial)`
   - Remove `Robot` table updates from RabbitMQ event handlers

3. **Update Usability Checks**:
   - Update `SupplyService.usabilityChecks` to use Device API response fields
   - Map Device API `OperationState` to internal `RobotStateEventState` enum

4. **Remove Event Processing**:
   - Remove `RobotEventsHandler.handleRobotStateChange()` RabbitMQ subscription
   - Remove robot state updates from `Resource` table

5. **Update Types**:
   - Replace `LegacyRobot` type with Device API `Device` type
   - Update `PlannerService` to use Device API types

#### Code Changes

**Before**:
```typescript
// supply.service.ts
async robots(): Promise<LegacyRobot[]> {
  const robots = await this.repo.getAll(ResourceType.Robot);
  return robots;
}
```

**After**:
```typescript
// supply.service.ts
async robots(): Promise<Device[]> {
  const response = await this.deviceApiClient.GetAvailableDevices({
    page_size: 1000,
  });
  return response.devices.map(device => this.mapDeviceToLegacyRobot(device));
}
```

**Files to Modify**:
- `delivery-platform/service/dispatch-engine/src/modules/supply/service/supply.service.ts`
- `delivery-platform/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts`
- `delivery-platform/service/dispatch-engine/src/shared/helpers/prisma-mappers.ts`
- `delivery-platform/lib/types/dispatch-engine/robots.ts`

### 2. Operations Service

#### Current Usage

Operations Service currently:
- Queries `RobotStateHistory` table for trip context state
- Uses `RobotEphemeralDataService` to get ephemeral data (location, battery, health)
- Caches ephemeral data in Redis (60-300 second TTL)
- Queries State Service via RPC for latest state
- Queries DynamoDB for device info

#### Migration Steps

1. **Add Device API Client**:
   - Add gRPC client dependency
   - Create `DeviceApiClient` service wrapper

2. **Replace Ephemeral Data Queries**:
   - Replace `RobotEphemeralDataService.getEphemeralDeviceInfo()` with `DeviceApiClient.GetDevice()` (robot telemetry)
   - Replace `RobotEphemeralDataService.getEphemeralState()` with `DeviceApiClient.GetDeviceState()`
   - Replace `RobotEphemeralDataService.getEphemeralHealthInfo()` with Device API `healthy` field (from robot telemetry)

3. **Update Robot Queries**:
   - Replace `RobotsService.get()` to use Device API for device info
   - Keep `RobotStateHistory` for historical queries (if needed)
   - Remove Redis caching of ephemeral data (Device API handles caching)

4. **Remove State Service RPC Calls**:
   - Remove `StateServiceRPC` calls
   - Remove DynamoDB queries for device info

5. **Update Operational Readiness Checks**:
   - Update `RobotsService.getRobotOperationalReadinessIssue()` to use Device API fields

#### Code Changes

**Before**:
```typescript
// robot-ephemeral-data.service.ts
async getEphemeralDeviceInfo(serial: string): Promise<RobotEphemeralDeviceInfo> {
  const cachedData = await this.cache.get<RobotEphemeralDeviceInfo>(key);
  if (cachedData != null) return cachedData;
  
  const latestRemoteState = await this.getLatestDeviceInfoFromDynamo(serial);
  // ... transform and cache
}
```

**After**:
```typescript
// robot-ephemeral-data.service.ts
async getEphemeralDeviceInfo(serial: string): Promise<RobotEphemeralDeviceInfo> {
  const device = await this.deviceApiClient.GetDevice({ 
    name: serial 
  });
  return this.mapDeviceToEphemeralDeviceInfo(device);
}
```

**Files to Modify**:
- `delivery-platform/service/operations/src/modules/robots/services/robot-ephemeral-data.service.ts`
- `delivery-platform/service/operations/src/modules/robots/services/robots.service.ts`
- `delivery-platform/service/operations/src/modules/robots/controllers/robots.controller.ts`

### 3. MDS Service

#### Current Usage

MDS Service currently:
- Queries Operations Service `/robots/{serial}/status` endpoint
- Implements `/v1/vehicles` endpoint (returns 200)
- **Does NOT implement** `/v1/vehicles/status` endpoint (returns 501)
- Transforms internal robot models to MDS format

#### Migration Steps

1. **Add Device API Client**:
   - Add gRPC client dependency (or use REST via gRPC-Gateway)
   - Create `DeviceApiClient` service wrapper

2. **Update Data Source**:
   - Replace Operations Service queries with Device API queries
   - Use `DeviceApiClient.ListDevices()` for bulk queries
   - Use `DeviceApiClient.GetDevice()` for single device queries
   - Use `DeviceApiClient.BatchGetDevices()` for batch queries

3. **Implement Missing Endpoints**:
   - Implement `GET /v1/vehicles/status` using Device API data
   - Transform Device API `Device` model to MDS `VehicleStatus` format
   - Map Device API `operation_state` to MDS `vehicle_state` enum
   - Map Device API `last_event` to MDS event format

4. **Update Existing Endpoints**:
   - Update `GET /v1/vehicles` to use `DeviceApiClient.ListDevices()`
   - Update `GET /v1/vehicles/:device_id` to use `DeviceApiClient.GetDevice()`
   - Ensure data freshness meets MDS requirements (Device API provides freshness guarantees)

5. **Remove Operations Service Dependency**:
   - Remove Operations Service HTTP client calls
   - Use Device API as single source of truth

6. **Add Rate Limiting**:
   - Implement rate limiting for external MDS consumers (within MDS Service)
   - Per-city API keys with different rate limits
   - DDoS protection (Cloudflare/AWS WAF)

#### Code Changes

**Before**:
```go
// mds_handler.go
func (s *MDSService) GetVehiclesStatus(w http.ResponseWriter, r *http.Request) {
    WriteError(ctx, w, NotImplemented(""))
}

// robots_manager.go
func (m *RobotsManager) GetRobotStatus(serial string) (*mdsmodels.VehicleStatus, error) {
    robot, err := m.robotsRepo.GetRobotFromOperations(serial)
    // ... transform to MDS format
}
```

**After**:
```go
// mds_handler.go
func (s *MDSService) GetVehiclesStatus(w http.ResponseWriter, r *http.Request) {
    devices, err := s.deviceApiClient.ListDevices(ctx, &deviceapi.ListDevicesRequest{
        PageSize: 1000,
    })
    if err != nil {
        WriteError(ctx, w, Internal("failed to get devices"), err)
        return
    }
    
    statuses := make([]mdsmodels.VehicleStatus, 0, len(devices.Devices))
    for _, device := range devices.Devices {
        status := s.transformDeviceToMDSStatus(device)
        statuses = append(statuses, *status)
    }
    
    WriteResponse(ctx, w, &mdsmodels.VehiclesStatusResponse{
        Version:        mdsmodels.DefaultVersion,
        LastUpdated:    time.Now().UTC().UnixMilli(),
        TTL:            0,
        VehiclesStatus: statuses,
    })
}

// robots_manager.go
func (m *RobotsManager) GetRobotStatus(serial string) (*mdsmodels.VehicleStatus, error) {
    device, err := m.deviceApiClient.GetDevice(ctx, &deviceapi.GetDeviceRequest{
        Name: fmt.Sprintf("devices/%s", serial),
    })
    if err != nil {
        return nil, err
    }
    return m.transformDeviceToMDSStatus(device), nil
}
```

**Files to Modify**:
- `coco-services/mds/internal/handlers/mds_handler.go`
- `coco-services/mds/internal/managers/robots_manager.go`
- `coco-services/mds/internal/repositories/robots/robots_operations.go`

### 4. Mission Control (Frontend)

#### Current Usage

Mission Control currently:
- Queries Operations Service `/devices` endpoints
- Uses `DeviceApi` class to fetch robot data
- Queries `/devices/state/latest` for latest state
- Queries `/devices/gps/batch` for geolocations
- Queries `/devices/:serial` for robot details

#### Migration Steps

1. **Update API Client**:
   - Add gRPC-Web client or REST client (via gRPC-Gateway)
   - Update `DeviceApi` class to use Device API endpoints

2. **Update Query Hooks**:
   - Update `useRobotDetails()` to use Device API
   - Update `useRobotLatestStates()` to use Device API
   - Update `fetchRobotsList()` to use Device API

3. **Update Type Definitions**:
   - Replace `DeviceRobotDetails` with Device API `Device` type
   - Replace `DeviceRobotState` with Device API `DeviceState` type

4. **Remove Old Endpoints**:
   - Remove calls to Operations Service `/devices/*` endpoints

#### Code Changes

**Before**:
```typescript
// device.api.ts
fetchRobotDetails({ serial }: FetchRobotDetailsParams): Promise<DeviceRobotDetails> {
  return this.apiClient.devices.get(`/devices/${serial}`);
}
```

**After**:
```typescript
// device.api.ts
fetchRobotDetails({ serial }: FetchRobotDetailsParams): Promise<Device> {
  return this.apiClient.deviceApi.get(`/v1/devices/${serial}`);
  // gRPC-Gateway automatically handles resource name format
}
```

**Files to Modify**:
- `delivery-platform/web/mission-control/src/api/device.api.ts`
- `delivery-platform/web/mission-control/src/features/device/api/robots/useRobotDetails.ts`
- `delivery-platform/web/mission-control/src/features/device/api/robots/useRobotLatestStates.ts`

## Implementation Phases

### Phase 1: Build Device API Service
- Create new service with gRPC server
- Implement state aggregation from State Service
- Implement caching layer
- Implement core gRPC endpoints
- Add monitoring and observability

### Phase 2: MDS Integration
- Migrate MDS Service to use Device API for data queries
- Ensure Device API provides all fields needed for MDS transformation
- Verify data freshness meets MDS requirements
- Test MDS compliance (MDS Service handles endpoint exposure and rate limiting)

### Phase 3: Internal Service Migration
- Migrate Dispatch Engine
- Migrate Operations Service
- Migrate Mission Control frontend
- Gradual rollout with feature flags

### Phase 4: Deprecation
- Deprecate old endpoints
- Remove duplicate event processing logic
- Consolidate state storage
- Remove unused code

## Dependencies

- **State Service**: Must continue processing events and publishing to RabbitMQ
- **RabbitMQ**: Must continue publishing events for backward compatibility
- **Operations Service**: Must maintain current APIs during migration
- **gRPC Infrastructure**: gRPC server setup, protobuf definitions, gRPC-Gateway (optional)

