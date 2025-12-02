# Device API: Implementation Milestones

## Overview

This document breaks down the Device API implementation into small, granular tasks that can be completed incrementally. Each task is designed to be:
- **Self-contained**: Can be implemented, reviewed, and committed independently
- **Testable**: Has clear acceptance criteria
- **Verifiable**: Includes specific commands/checks to confirm completion
- **Context-aware**: References relevant documentation for new agents

## Context for New Agents

If you're picking up this work mid-implementation, please read:
1. **Product Requirements**: `obsidian/designs/Device API - Product Requirements Document.md`
2. **Technical Design**: `obsidian/designs/Device API - Technical Design.md`
3. **This Document**: Check completed tasks (✅) to see what's already done

## Project Structure

The Device API is a **Go service** located at:
- **Service Path**: `coco-services/device-api/`
- **Proto Path**: `coco-services/device-api/proto/deviceapi/v1/device_api.proto`
- **Main Entry**: `coco-services/device-api/cmd/server.go`

---

## Milestone 1: Project Setup & Infrastructure

### Task 1.1: Create Service Directory Structure
**Status**: ⬜ Not Started  
**Dependencies**: None  
**Estimated Time**: 15 minutes

**Description**: Create the basic directory structure for the new Device API service following Go project conventions.

**Acceptance Criteria**:
- [ ] Directory `coco-services/device-api/` exists
- [ ] Subdirectories created: `cmd/`, `internal/`, `proto/`, `pkg/`
- [ ] `go.mod` file created with module name
- [ ] Basic `Makefile` created

**Files to Create**:
- `coco-services/device-api/go.mod`
- `coco-services/device-api/Makefile`
- `coco-services/device-api/cmd/.gitkeep`
- `coco-services/device-api/internal/.gitkeep`
- `coco-services/device-api/proto/.gitkeep`

**Verification**:
```bash
cd coco-services/device-api
# Verify directory structure exists
ls -la cmd/ internal/ proto/ pkg/
# Verify go.mod is valid
cat go.mod | head -5
# Should show: module github.com/cocorobotics/coco-services/device-api
```

**Reference**: Look at `coco-services/fleet/` or `coco-services/mds/` for structure patterns.

---

### Task 1.2: Configure go.mod with Dependencies
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.1  
**Estimated Time**: 20 minutes

**Description**: Set up `go.mod` with required dependencies for gRPC, database access, Redis, and RabbitMQ.

**Acceptance Criteria**:
- [ ] gRPC dependencies (`google.golang.org/grpc`, `google.golang.org/protobuf`)
- [ ] Database drivers (PostgreSQL for State/Operations DBs)
- [ ] Redis client (`github.com/go-redis/redis/v8`)
- [ ] RabbitMQ client (`github.com/rabbitmq/amqp091-go`)
- [ ] Shared packages from `coco-services/core/`
- [ ] Datadog tracing (`gopkg.in/DataDog/dd-trace-go.v1`)

**Files to Modify**:
- `coco-services/device-api/go.mod`

**Verification**:
```bash
cd coco-services/device-api
# Download dependencies and verify they resolve
go mod download
go mod tidy
# Should complete without errors
echo $?  # Should be 0
```

**Reference**: 
- Check `coco-services/fleet/go.mod` for dependency patterns
- Check `coco-services/mds/go.mod` for MDS Service dependencies

---

### Task 1.3: Create Main Entry Point and Config
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.2  
**Estimated Time**: 30 minutes

**Description**: Create the main entry point with Cobra commands and configuration loading.

**Acceptance Criteria**:
- [ ] `cmd/root.go` with Cobra root command
- [ ] `cmd/server.go` with server command
- [ ] `internal/config/config.go` for configuration struct
- [ ] Environment variable loading (database URLs, Redis, RabbitMQ, ports)
- [ ] Service can start without errors (even if it doesn't do anything yet)

**Files to Create**:
- `coco-services/device-api/cmd/root.go`
- `coco-services/device-api/cmd/server.go`
- `coco-services/device-api/internal/config/config.go`
- `coco-services/device-api/main.go`

**Verification**:
```bash
cd coco-services/device-api
# Build the binary
go build -o device-api .
# Should compile without errors
echo $?  # Should be 0
# Verify help command works
./device-api --help
# Should show available commands including "server"
./device-api server --help
# Should show server command options
```

**Reference**: 
- `coco-services/fleet/cmd/server.go`
- `coco-services/mds/cmd/app.go`

---

### Task 1.4: Set Up Logging and Tracing
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.3  
**Estimated Time**: 20 minutes

**Description**: Configure structured logging and Datadog tracing.

**Acceptance Criteria**:
- [ ] Structured logger configured (use `coco-services/core/log` package)
- [ ] Datadog tracer initialized
- [ ] Context propagation for traces
- [ ] Log levels configurable via environment

**Files to Modify**:
- `coco-services/device-api/cmd/server.go`

**Verification**:
```bash
cd coco-services/device-api
# Build and run with logging enabled
go build -o device-api .
LOG_LEVEL=debug ./device-api server &
# Should see structured log output (JSON format)
# Kill the process after verifying
kill %1
```

**Reference**: 
- `coco-services/fleet/cmd/server.go` (tracer setup)
- `coco-services/core/log/` (logging package)

---

## Milestone 2: Protobuf Definitions & Code Generation

### Task 2.1: Create Protobuf Service Definition
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.1  
**Estimated Time**: 45 minutes

**Description**: Create the `.proto` file defining the Device API gRPC service and all message types per the Technical Design.

**Acceptance Criteria**:
- [ ] `proto/deviceapi/v1/device_api.proto` file created
- [ ] `DeviceService` service defined with all RPC methods:
  - `GetDevice`
  - `GetDeviceState`
  - `ListDevices`
  - `BatchGetDevices`
  - `GetAvailableDevices`
  - `StreamDeviceUpdates`
- [ ] All request/response messages defined per Technical Design
- [ ] All enums defined (`OperationState`, `TripType`, `LocationType`)
- [ ] All nested messages defined (`Device`, `Location`, `Battery`, `Connectivity`, `DriveUStatus`, etc.)
- [ ] Follows AIP conventions (resource names, field masks, pagination)

**Files to Create**:
- `coco-services/device-api/proto/deviceapi/v1/device_api.proto`

**Verification**:
```bash
cd coco-services/device-api
# Verify proto file syntax is valid (requires protoc installed)
protoc --proto_path=proto proto/deviceapi/v1/device_api.proto --descriptor_set_out=/dev/null
# Should complete without errors
echo $?  # Should be 0
# Or just verify the file exists and has expected content
grep "service DeviceService" proto/deviceapi/v1/device_api.proto
grep "rpc GetDevice" proto/deviceapi/v1/device_api.proto
grep "message Device" proto/deviceapi/v1/device_api.proto
```

**Reference**: 
- Technical Design: `obsidian/designs/Device API - Technical Design.md` (protobuf section)
- Example proto: `coco-services/fleet/proto/` (for patterns)

---

### Task 2.2: Set Up Protobuf Code Generation
**Status**: ⬜ Not Started  
**Dependencies**: Task 2.1  
**Estimated Time**: 30 minutes

**Description**: Configure build scripts to generate Go code from `.proto` files.

**Acceptance Criteria**:
- [ ] `buf.yaml` or protoc configuration set up
- [ ] Makefile target for proto generation (`make proto`)
- [ ] Generated Go code output to `pkg/deviceapi/v1/`
- [ ] Generated code compiles without errors

**Files to Create/Modify**:
- `coco-services/device-api/Makefile` (add proto target)
- `coco-services/device-api/buf.yaml` (if using buf) or proto generation script

**Verification**:
```bash
cd coco-services/device-api
# Run proto generation
make proto
# Should complete without errors
echo $?  # Should be 0
# Verify generated files exist
ls pkg/deviceapi/v1/*.pb.go
# Should list: device_api.pb.go, device_api_grpc.pb.go
```

**Reference**: 
- Check how Fleet Service generates proto code

---

### Task 2.3: Verify Protobuf Compilation
**Status**: ⬜ Not Started  
**Dependencies**: Task 2.2  
**Estimated Time**: 15 minutes

**Description**: Run proto compilation and verify generated code is correct.

**Acceptance Criteria**:
- [ ] `make proto` runs without errors
- [ ] Generated Go files exist in `pkg/deviceapi/v1/`
- [ ] All service methods and message types are available
- [ ] Can import generated types in a test file

**Files to Verify**:
- `coco-services/device-api/pkg/deviceapi/v1/*.pb.go`

**Verification**:
```bash
cd coco-services/device-api
# Regenerate to ensure clean state
make proto
# Build the entire project to verify generated code compiles
go build ./...
# Should complete without errors
echo $?  # Should be 0
# Verify key types exist in generated code
grep "type Device struct" pkg/deviceapi/v1/device_api.pb.go
grep "type DeviceServiceServer interface" pkg/deviceapi/v1/device_api_grpc.pb.go
```

---

## Milestone 3: Database & External Service Connections

### Task 3.1: Set Up State Service Database Connection
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.4  
**Estimated Time**: 30 minutes

**Description**: Create database connection to State Service database (read replica) for querying robot telemetry.

**Acceptance Criteria**:
- [ ] PostgreSQL connection configured for State Service DB
- [ ] Connection pool settings configured
- [ ] Can query `StateHistory` table
- [ ] Read replica URL configurable via environment

**Files to Create**:
- `coco-services/device-api/internal/repositories/state/state_repo.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests for the repository (if created)
go test ./internal/repositories/state/... -v
# For integration test (requires DB connection):
# STATE_DB_URL="postgres://..." go test ./internal/repositories/state/... -v -tags=integration
```

**Reference**: 
- Check State Service Prisma schema for table structure

---

### Task 3.2: Set Up Operations Service Database Connection
**Status**: ⬜ Not Started  
**Dependencies**: Task 3.1  
**Estimated Time**: 30 minutes

**Description**: Create database connection to Operations Service database for querying trip context.

**Acceptance Criteria**:
- [ ] PostgreSQL connection configured for Operations Service DB
- [ ] Can query `RobotStateHistory` table
- [ ] Connection URL configurable via environment

**Files to Create**:
- `coco-services/device-api/internal/repositories/operations/operations_repo.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests
go test ./internal/repositories/operations/... -v
```

**Reference**: 
- Check Operations Service Prisma schema for `RobotStateHistory` table

---

### Task 3.3: Set Up Redis Cache
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.4  
**Estimated Time**: 30 minutes

**Description**: Configure Redis connection for caching device state.

**Acceptance Criteria**:
- [ ] Redis client initialized (`go-redis/redis`)
- [ ] Connection tested (can set/get a test value)
- [ ] Connection URL configurable via environment
- [ ] TTL support verified

**Files to Create**:
- `coco-services/device-api/internal/cache/cache.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests with mock Redis
go test ./internal/cache/... -v
# For integration test (requires Redis):
# REDIS_URL="redis://localhost:6379" go test ./internal/cache/... -v -tags=integration
```

**Reference**: 
- Check how Fleet Service or MDS Service uses Redis (if applicable)

---

### Task 3.4: Set Up RabbitMQ Consumer
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.4  
**Estimated Time**: 30 minutes

**Description**: Configure RabbitMQ connection for subscribing to device state events.

**Acceptance Criteria**:
- [ ] RabbitMQ connection established
- [ ] Can subscribe to a test exchange/queue
- [ ] Connection URL configurable via environment
- [ ] Reconnection logic implemented

**Files to Create**:
- `coco-services/device-api/internal/consumer/rabbitmq.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests
go test ./internal/consumer/... -v
```

**Reference**: 
- Check how other Go services consume RabbitMQ events

---

### Task 3.5: Set Up Fleet Service gRPC Client
**Status**: ⬜ Not Started  
**Dependencies**: Task 1.4  
**Estimated Time**: 30 minutes

**Description**: Create gRPC client to query Fleet Service for deployment information.

**Acceptance Criteria**:
- [ ] Fleet Service gRPC client created
- [ ] Can make test request to Fleet Service
- [ ] Client configuration in config
- [ ] Error handling for Fleet Service unavailability

**Files to Create**:
- `coco-services/device-api/internal/clients/fleet_client.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests with mock gRPC server
go test ./internal/clients/... -v
```

**Reference**: 
- Fleet Service proto definitions

---

## Milestone 4: State Aggregation Logic

### Task 4.1: Create Device Aggregation Service
**Status**: ⬜ Not Started  
**Dependencies**: Task 3.1, Task 3.2, Task 3.5  
**Estimated Time**: 30 minutes

**Description**: Create the main service for aggregating device state from multiple sources.

**Acceptance Criteria**:
- [ ] `DeviceAggregator` struct created
- [ ] Method: `AggregateDeviceState(ctx, deviceID) (*pb.Device, error)`
- [ ] Dependencies injected (repos, clients)
- [ ] Placeholder implementation (returns mock data)

**Files to Create**:
- `coco-services/device-api/internal/service/aggregator.go`

**Verification**:
```bash
cd coco-services/device-api
# Build to verify code compiles
go build ./...
echo $?  # Should be 0
# Run unit tests with mocks
go test ./internal/service/... -v -run TestAggregator
```

---

### Task 4.2: Implement Robot Telemetry Aggregation
**Status**: ⬜ Not Started  
**Dependencies**: Task 4.1  
**Estimated Time**: 1 hour

**Description**: Implement logic to fetch robot telemetry from State Service DB.

**Acceptance Criteria**:
- [ ] Queries `StateHistory` table for latest operation state
- [ ] Queries latest heartbeat data (location, battery, connectivity)
- [ ] Queries DriveU status
- [ ] Maps database fields to protobuf message fields
- [ ] Handles missing data gracefully

**Files to Modify**:
- `coco-services/device-api/internal/service/aggregator.go`
- `coco-services/device-api/internal/repositories/state/state_repo.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run unit tests for telemetry aggregation
go test ./internal/service/... -v -run TestTelemetry
go test ./internal/repositories/state/... -v
```

---

### Task 4.3: Implement Trip Context Aggregation
**Status**: ⬜ Not Started  
**Dependencies**: Task 4.2  
**Estimated Time**: 45 minutes

**Description**: Implement logic to fetch trip context from Operations Service DB.

**Acceptance Criteria**:
- [ ] Queries `RobotStateHistory` table for latest trip context
- [ ] Maps fields: `needsMovement`, `hasFood`, `tripType`, etc.
- [ ] Handles missing data (optional fields)
- [ ] Maps internal enums to protobuf enums

**Files to Modify**:
- `coco-services/device-api/internal/service/aggregator.go`
- `coco-services/device-api/internal/repositories/operations/operations_repo.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run unit tests for trip context
go test ./internal/service/... -v -run TestTripContext
go test ./internal/repositories/operations/... -v
```

---

### Task 4.4: Implement Fleet Assignment Aggregation
**Status**: ⬜ Not Started  
**Dependencies**: Task 4.3  
**Estimated Time**: 30 minutes

**Description**: Implement logic to fetch fleet assignment from Fleet Service.

**Acceptance Criteria**:
- [ ] Calls Fleet Service gRPC API for deployment info
- [ ] Maps response to `Deployment` protobuf message
- [ ] Handles Fleet Service unavailability (returns nil)
- [ ] Maps location types correctly

**Files to Modify**:
- `coco-services/device-api/internal/service/aggregator.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run unit tests for fleet assignment (with mock Fleet client)
go test ./internal/service/... -v -run TestFleetAssignment
```

---

### Task 4.5: Implement Parallel Aggregation
**Status**: ⬜ Not Started  
**Dependencies**: Task 4.4  
**Estimated Time**: 30 minutes

**Description**: Use `errgroup` to query all data sources in parallel.

**Acceptance Criteria**:
- [ ] Parallel queries using `golang.org/x/sync/errgroup`
- [ ] Context cancellation propagated
- [ ] Partial failures handled (return partial data if possible)
- [ ] Metadata fields calculated (`update_time`, `last_synced`, `data_freshness_seconds`)

**Files to Modify**:
- `coco-services/device-api/internal/service/aggregator.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run full aggregation tests
go test ./internal/service/... -v -run TestAggregateDeviceState
# Verify parallel execution with race detector
go test ./internal/service/... -v -race
```

---

### Task 4.6: Add Unit Tests for Aggregation Logic
**Status**: ⬜ Not Started  
**Dependencies**: Task 4.5  
**Estimated Time**: 1 hour

**Description**: Write unit tests for state aggregation service.

**Acceptance Criteria**:
- [ ] Tests for `AggregateDeviceState` method
- [ ] Tests for each data source (telemetry, trip context, fleet assignment)
- [ ] Tests for missing data handling
- [ ] Tests for enum mapping
- [ ] Mocks for repositories and clients

**Files to Create**:
- `coco-services/device-api/internal/service/aggregator_test.go`

**Verification**:
```bash
cd coco-services/device-api
# Run all aggregator tests
go test ./internal/service/... -v
# Check test coverage
go test ./internal/service/... -cover
# Coverage should be > 80%
```

---

## Milestone 5: Caching Layer

### Task 5.1: Implement Cache Service
**Status**: ⬜ Not Started  
**Dependencies**: Task 3.3  
**Estimated Time**: 30 minutes

**Description**: Create cache service for device state caching.

**Acceptance Criteria**:
- [ ] `DeviceCache` struct created
- [ ] Methods: `Get(deviceID) (*pb.Device, error)`
- [ ] Methods: `Set(deviceID, device, ttl) error`
- [ ] Methods: `Delete(deviceID) error`
- [ ] Uses protobuf serialization for cache values

**Files to Create**:
- `coco-services/device-api/internal/cache/device_cache.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run cache unit tests
go test ./internal/cache/... -v -run TestDeviceCache
```

---

### Task 5.2: Integrate Caching into Aggregation
**Status**: ⬜ Not Started  
**Dependencies**: Task 5.1, Task 4.5  
**Estimated Time**: 30 minutes

**Description**: Add caching layer to state aggregation service.

**Acceptance Criteria**:
- [ ] Check cache before querying databases
- [ ] Return cached data if available
- [ ] Query databases on cache miss
- [ ] Store result in cache with 5-second TTL
- [ ] Handle cache errors gracefully

**Files to Modify**:
- `coco-services/device-api/internal/service/aggregator.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run tests that verify caching behavior
go test ./internal/service/... -v -run TestCaching
# Test should verify:
# 1. First call hits DB
# 2. Second call returns cached data
# 3. After TTL, call hits DB again
```

---

### Task 5.3: Implement Cache Invalidation on Events
**Status**: ⬜ Not Started  
**Dependencies**: Task 5.2, Task 3.4  
**Estimated Time**: 45 minutes

**Description**: Subscribe to RabbitMQ events and invalidate cache on device state changes.

**Acceptance Criteria**:
- [ ] RabbitMQ consumer handler created
- [ ] Subscribes to: `IoT.Heartbeat`, `IoT.HealthStateChanged`, `Robots.StateChange`, `ConnectivityRawStatusChanged`
- [ ] Extracts device ID from event
- [ ] Invalidates cache for that device
- [ ] Handles event parsing errors

**Files to Create**:
- `coco-services/device-api/internal/consumer/device_events.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Run event handler tests
go test ./internal/consumer/... -v -run TestDeviceEvents
# Test should verify:
# 1. Event received -> cache invalidated
# 2. Malformed event -> error logged, no crash
```

---

### Task 5.4: Add Cache Tests
**Status**: ⬜ Not Started  
**Dependencies**: Task 5.3  
**Estimated Time**: 30 minutes

**Description**: Write unit tests for caching logic.

**Acceptance Criteria**:
- [ ] Tests for cache hit/miss scenarios
- [ ] Tests for cache invalidation
- [ ] Tests for TTL expiration
- [ ] Tests for cache error handling

**Files to Create**:
- `coco-services/device-api/internal/cache/device_cache_test.go`
- `coco-services/device-api/internal/consumer/device_events_test.go`

**Verification**:
```bash
cd coco-services/device-api
# Run all cache-related tests
go test ./internal/cache/... ./internal/consumer/... -v
# Check coverage
go test ./internal/cache/... ./internal/consumer/... -cover
# Coverage should be > 80%
```

---

## Milestone 6: gRPC Service Implementation

### Task 6.1: Create gRPC Server Setup
**Status**: ⬜ Not Started  
**Dependencies**: Task 2.2  
**Estimated Time**: 30 minutes

**Description**: Configure gRPC server with interceptors and health checks.

**Acceptance Criteria**:
- [ ] gRPC server created with Datadog tracing interceptor
- [ ] Health check service registered
- [ ] Reflection enabled for debugging
- [ ] Server starts on configured port

**Files to Modify**:
- `coco-services/device-api/cmd/server.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
# Start server in background
GRPC_PORT=50051 ./device-api server &
sleep 2
# Test with grpcurl (requires grpcurl installed)
grpcurl -plaintext localhost:50051 list
# Should show: deviceapi.v1.DeviceService, grpc.health.v1.Health
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
# Should return: serving: SERVING
# Cleanup
kill %1
```

**Reference**: 
- `coco-services/fleet/cmd/server.go` (gRPC setup)

---

### Task 6.2: Create gRPC Handler Struct
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.1, Task 4.5  
**Estimated Time**: 20 minutes

**Description**: Create the handler struct implementing `DeviceServiceServer`.

**Acceptance Criteria**:
- [ ] `DeviceServer` struct created
- [ ] Implements `pb.DeviceServiceServer` interface
- [ ] Dependencies injected (aggregator, cache)
- [ ] All methods defined as stubs

**Files to Create**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build ./...
echo $?  # Should be 0
# Verify interface is fully implemented (no compile errors)
# The build would fail if DeviceServiceServer interface is not satisfied
```

---

### Task 6.3: Implement GetDevice RPC
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.2, Task 5.2  
**Estimated Time**: 45 minutes

**Description**: Implement `GetDevice` RPC method.

**Acceptance Criteria**:
- [ ] Parses resource name (`devices/{device_id}`)
- [ ] Validates device ID format
- [ ] Calls aggregator (which uses cache)
- [ ] Applies field mask if provided
- [ ] Returns `NOT_FOUND` if device doesn't exist
- [ ] Returns `INVALID_ARGUMENT` for malformed input

**Files to Modify**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
# Start server with mock data
./device-api server &
sleep 2
# Test GetDevice with grpcurl
grpcurl -plaintext -d '{"name": "devices/C10190"}' localhost:50051 deviceapi.v1.DeviceService/GetDevice
# Should return device data
# Test invalid resource name
grpcurl -plaintext -d '{"name": "invalid"}' localhost:50051 deviceapi.v1.DeviceService/GetDevice
# Should return INVALID_ARGUMENT error
kill %1
```

---

### Task 6.4: Implement GetDeviceState RPC
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.3  
**Estimated Time**: 30 minutes

**Description**: Implement lightweight `GetDeviceState` method.

**Acceptance Criteria**:
- [ ] Returns only state summary fields
- [ ] Uses same aggregator but filters response
- [ ] Applies field mask if provided

**Files to Modify**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
grpcurl -plaintext -d '{"name": "devices/C10190"}' localhost:50051 deviceapi.v1.DeviceService/GetDeviceState
# Should return only state fields (operation_state, connectivity, healthy, etc.)
kill %1
```

---

### Task 6.5: Implement ListDevices RPC
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.4  
**Estimated Time**: 1.5 hours

**Description**: Implement `ListDevices` with filtering and pagination.

**Acceptance Criteria**:
- [ ] Supports all filters: `healthy`, `operation_state`, `deployed_location_id`, etc.
- [ ] Implements token-based pagination
- [ ] Default `page_size` = 50, max = 1000
- [ ] Returns `next_page_token` for pagination
- [ ] Handles invalid filters

**Files to Modify**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
# Test list with pagination
grpcurl -plaintext -d '{"page_size": 10}' localhost:50051 deviceapi.v1.DeviceService/ListDevices
# Should return devices array and next_page_token
# Test with filter
grpcurl -plaintext -d '{"healthy": true}' localhost:50051 deviceapi.v1.DeviceService/ListDevices
# Should return only healthy devices
kill %1
```

---

### Task 6.6: Implement BatchGetDevices RPC
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.5  
**Estimated Time**: 45 minutes

**Description**: Implement batch get for multiple devices.

**Acceptance Criteria**:
- [ ] Accepts array of resource names
- [ ] Validates all resource names
- [ ] Batch queries databases efficiently
- [ ] Returns devices in same order as requested
- [ ] Omits missing devices

**Files to Modify**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
grpcurl -plaintext -d '{"names": ["devices/C10190", "devices/C10191"]}' localhost:50051 deviceapi.v1.DeviceService/BatchGetDevices
# Should return array of devices in same order
kill %1
```

---

### Task 6.7: Implement GetAvailableDevices RPC
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.6  
**Estimated Time**: 45 minutes

**Description**: Implement custom method for Dispatch Engine.

**Acceptance Criteria**:
- [ ] Filters for available robots (PARKED, healthy, online)
- [ ] Supports location filters
- [ ] Supports pagination

**Files to Modify**:
- `coco-services/device-api/internal/handlers/grpc/device_handler.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
grpcurl -plaintext -d '{}' localhost:50051 deviceapi.v1.DeviceService/GetAvailableDevices
# Should return only available devices (PARKED, healthy, online)
kill %1
```

---

### Task 6.8: Add Error Handling
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.7  
**Estimated Time**: 30 minutes

**Description**: Implement proper gRPC error handling per AIP-193.

**Acceptance Criteria**:
- [ ] Returns `NOT_FOUND` for missing devices
- [ ] Returns `INVALID_ARGUMENT` for invalid parameters
- [ ] Returns `UNAVAILABLE` for database errors
- [ ] Returns `INTERNAL` for unexpected errors
- [ ] Error messages are human-readable

**Files to Create/Modify**:
- `coco-services/device-api/internal/handlers/grpc/errors.go`

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
# Test NOT_FOUND
grpcurl -plaintext -d '{"name": "devices/NONEXISTENT"}' localhost:50051 deviceapi.v1.DeviceService/GetDevice 2>&1
# Should contain "NOT_FOUND"
# Test INVALID_ARGUMENT
grpcurl -plaintext -d '{"name": ""}' localhost:50051 deviceapi.v1.DeviceService/GetDevice 2>&1
# Should contain "INVALID_ARGUMENT"
kill %1
```

---

### Task 6.9: Add gRPC Handler Tests
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.8  
**Estimated Time**: 1.5 hours

**Description**: Write tests for gRPC handlers.

**Acceptance Criteria**:
- [ ] Tests for each RPC method
- [ ] Tests for error cases
- [ ] Tests for pagination
- [ ] Tests for field masks

**Files to Create**:
- `coco-services/device-api/internal/handlers/grpc/device_handler_test.go`

**Verification**:
```bash
cd coco-services/device-api
# Run all handler tests
go test ./internal/handlers/grpc/... -v
# Check coverage
go test ./internal/handlers/grpc/... -cover
# Coverage should be > 80%
```

---

## Milestone 7: Integration & Testing

### Task 7.1: Set Up Integration Test Environment
**Status**: ⬜ Not Started  
**Dependencies**: Task 6.9  
**Estimated Time**: 30 minutes

**Description**: Configure integration test setup.

**Acceptance Criteria**:
- [ ] Test database setup
- [ ] Mock services for external dependencies
- [ ] Test data fixtures

**Files to Create**:
- `coco-services/device-api/test/setup_test.go`
- `coco-services/device-api/test/helpers.go`

**Verification**:
```bash
cd coco-services/device-api
# Run integration test setup
go test ./test/... -v -run TestSetup
# Should complete without errors
```

---

### Task 7.2: End-to-End Integration Test
**Status**: ⬜ Not Started  
**Dependencies**: Task 7.1  
**Estimated Time**: 1 hour

**Description**: Write end-to-end tests.

**Acceptance Criteria**:
- [ ] Test: Event → Cache invalidation → Query → Fresh data
- [ ] Test: Query → Cache populated → Query again → Cached data
- [ ] Test: Batch queries work correctly

**Files to Create**:
- `coco-services/device-api/test/e2e_test.go`

**Verification**:
```bash
cd coco-services/device-api
# Run integration tests (requires Docker for dependencies)
docker-compose -f test/docker-compose.yml up -d
go test ./test/... -v -tags=integration
docker-compose -f test/docker-compose.yml down
```

---

## Milestone 8: Consumer Migration - Dispatch Engine

### Task 8.1: Create Device API gRPC Client for Dispatch Engine
**Status**: ⬜ Not Started  
**Dependencies**: Milestone 6 complete  
**Estimated Time**: 30 minutes

**Description**: Create gRPC client in Dispatch Engine (TypeScript) to call Device API.

**Acceptance Criteria**:
- [ ] gRPC client configured with Device API endpoint
- [ ] TypeScript types generated from proto
- [ ] Methods: `getDevice()`, `getAvailableDevices()`, `batchGetDevices()`

**Files to Create**:
- `delivery-platform/service/dispatch-engine/src/modules/device-api/device-api-client.service.ts`
- `delivery-platform/service/dispatch-engine/src/modules/device-api/device-api.module.ts`

**Verification**:
```bash
cd delivery-platform/service/dispatch-engine
# Build to verify code compiles
yarn build
echo $?  # Should be 0
# Run unit tests
yarn test --grep "DeviceApiClient"
```

---

### Task 8.2: Update Supply Service
**Status**: ⬜ Not Started  
**Dependencies**: Task 8.1  
**Estimated Time**: 1 hour

**Description**: Replace robot queries with Device API calls.

**Acceptance Criteria**:
- [ ] `SupplyService.robots()` uses Device API
- [ ] Maps Device API response to internal types
- [ ] Tests updated

**Files to Modify**:
- `delivery-platform/service/dispatch-engine/src/modules/supply/service/supply.service.ts`

**Verification**:
```bash
cd delivery-platform/service/dispatch-engine
yarn build
yarn test --grep "SupplyService"
# All tests should pass
```

---

### Task 8.3: Remove Robot Event Processing
**Status**: ⬜ Not Started  
**Dependencies**: Task 8.2  
**Estimated Time**: 30 minutes

**Description**: Remove RabbitMQ handlers that update robot state.

**Acceptance Criteria**:
- [ ] Remove `handleRobotStateChange()` subscription
- [ ] Remove robot state updates from tables

**Files to Modify**:
- `delivery-platform/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts`

**Verification**:
```bash
cd delivery-platform/service/dispatch-engine
yarn build
yarn test
# All tests should pass, no robot event handler tests should exist
```

---

## Milestone 9: Consumer Migration - Operations Service

### Task 9.1: Create Device API Client for Operations
**Status**: ⬜ Not Started  
**Dependencies**: Milestone 6 complete  
**Estimated Time**: 30 minutes

**Description**: Create gRPC client in Operations Service.

**Files to Create**:
- `delivery-platform/service/operations/src/modules/device-api/device-api-client.service.ts`

**Verification**:
```bash
cd delivery-platform/service/operations
yarn build
yarn test --grep "DeviceApiClient"
```

---

### Task 9.2: Replace Ephemeral Data Service
**Status**: ⬜ Not Started  
**Dependencies**: Task 9.1  
**Estimated Time**: 1 hour

**Description**: Update `RobotEphemeralDataService` to use Device API.

**Files to Modify**:
- `delivery-platform/service/operations/src/modules/robots/services/robot-ephemeral-data.service.ts`

**Verification**:
```bash
cd delivery-platform/service/operations
yarn build
yarn test --grep "RobotEphemeralDataService"
# All tests should pass
```

---

## Milestone 10: Consumer Migration - MDS Service

### Task 10.1: Create Device API Client for MDS
**Status**: ⬜ Not Started  
**Dependencies**: Milestone 6 complete  
**Estimated Time**: 30 minutes

**Description**: Create gRPC client in MDS Service (Go).

**Files to Create**:
- `coco-services/mds/internal/clients/device_api_client.go`

**Verification**:
```bash
cd coco-services/mds
go build ./...
go test ./internal/clients/... -v -run TestDeviceApiClient
```

---

### Task 10.2: Update MDS Robots Manager
**Status**: ⬜ Not Started  
**Dependencies**: Task 10.1  
**Estimated Time**: 1 hour

**Description**: Replace Operations Service queries with Device API.

**Files to Modify**:
- `coco-services/mds/internal/managers/robots_manager.go`

**Verification**:
```bash
cd coco-services/mds
go build ./...
go test ./internal/managers/... -v -run TestRobotsManager
```

---

## Milestone 11: Documentation & Deployment

### Task 11.1: Write README
**Status**: ⬜ Not Started  
**Dependencies**: Milestone 6 complete  
**Estimated Time**: 30 minutes

**Description**: Document the service.

**Files to Create**:
- `coco-services/device-api/README.md`

**Verification**:
```bash
cd coco-services/device-api
# Verify README exists and has key sections
grep "## Overview" README.md
grep "## Running" README.md
grep "## API" README.md
```

---

### Task 11.2: Add Monitoring & Health Checks
**Status**: ⬜ Not Started  
**Dependencies**: Task 11.1  
**Estimated Time**: 30 minutes

**Description**: Add metrics and health endpoints.

**Acceptance Criteria**:
- [ ] Prometheus metrics
- [ ] Health check endpoint
- [ ] Datadog integration

**Verification**:
```bash
cd coco-services/device-api
go build -o device-api .
./device-api server &
sleep 2
# Test health endpoint
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
# Test metrics endpoint (if HTTP)
curl localhost:8080/metrics
# Should return Prometheus metrics
kill %1
```

---

### Task 11.3: Create Deployment Configuration
**Status**: ⬜ Not Started  
**Dependencies**: Task 11.2  
**Estimated Time**: 30 minutes

**Description**: Create Docker/K8s configuration.

**Files to Create**:
- `coco-services/device-api/Dockerfile`

**Verification**:
```bash
cd coco-services/device-api
# Build Docker image
docker build -t device-api:test .
# Should complete without errors
echo $?  # Should be 0
# Verify image runs
docker run --rm device-api:test --help
# Should show help output
```

---

## Progress Tracking

### Completed Tasks
(Agent: Mark tasks as ✅ when completed and committed)

### Current Task
(Agent: Update this when starting a new task)

### Next Task
(Agent: Update this when current task is complete)

---

## Notes for AI Agents

1. **Always check completed tasks** before starting work
2. **Read the Technical Design** for detailed specifications
3. **Reference existing Go services** for patterns (Fleet, MDS)
4. **Run verification steps** before marking task complete
5. **Ask for review** after completing each task
6. **Commit incrementally** - one task per commit when possible
7. **Update this document** with task status as you progress

---

## Quick Reference

- **PRD**: `obsidian/designs/Device API - Product Requirements Document.md`
- **Technical Design**: `obsidian/designs/Device API - Technical Design.md`
- **Service Path**: `coco-services/device-api/`
- **Proto Path**: `coco-services/device-api/proto/deviceapi/v1/device_api.proto`
