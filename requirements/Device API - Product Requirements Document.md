# Device API: Product Requirements Document

# Problem Statement

Multiple services independently process MQTT/IoT events from robots, leading to inconsistent device state across our platform. There is no single source of truth for robot state, making it difficult to:

- Get accurate, up-to-date robot information
- Debug issues when different services show different states
- Provide useful monitoring information to Mission Control UI (data is currently scattered across multiple services)
- Add new consumers (like MDS endpoints) without duplicating event processing logic
- Ensure all services make decisions based on the same data

## Current State

### 1. Multiple Sources of Truth

Different services maintain their own versions of robot state:

- **State Service**: Processes events and stores state transitions
- **Operations Service**: Maintains business state in database + Redis cache
- **Dispatch Engine**: Stores robot state in its own tables
- **Fleet Service**: Stores robot state in DynamoDB
- **MDS Service**: Queries Operations service for status (will migrate to Device API)

**Impact**: Services see different states for the same robot at the same time, leading to operational errors.

### 2. Inconsistent Data Freshness

Services have different freshness guarantees:

- Some use Redis cache with 60-300 second TTLs
- Some process events in real-time but may miss events if consumers fail
- Some query databases that may be stale

**Impact**: Services make decisions based on outdated information, possibly causing incorrect assignments and failed deliveries.

### 3. Schema Inconsistencies

Different services use different data structures for the same information:

- Operation states use different enums (`'GROUNDED'` vs `RobotStateEventState` vs `RobotOperationState` )
- Location fields vary (`latitude/longitude` vs `lat/lng`)
- Battery data represented differently (`soc` vs `currentBatteryPercentage`)

**Impact**: Integration bugs, mapping errors, and confusion about which fields to use.

### 4. Mission Control UI Monitoring Gap

Mission Control UI needs comprehensive device information for monitoring and debugging, but currently must query multiple services and endpoints to get complete robot state:

- Operations Service `/devices` endpoints for basic device info
- Operations Service `/devices/state/latest` for state history
- Operations Service `/devices/gps/batch` for geolocations
- State Service for connectivity and state transitions
- Fleet Service for deployment information

**Impact**: Mission Control UI cannot efficiently display a unified view of robot state, making monitoring and debugging difficult. Engineers must manually correlate data from multiple sources when investigating issues.

## Requirements

### 1. Single Source of Truth

A unified Device API that serves as the authoritative source for all robot/device state information. All services should query this API instead of processing events independently. This includes robot telemetry (location, battery, operation state), connectivity status (including DriveU remote control status), trip context, and deployment information.

### 2. Consistent State

All services should see the same state for the same robot at the same time (within acceptable freshness windows).

### 3. Standardized Schema

A single, well-documented schema for device state that all consumers use, eliminating mapping inconsistencies.

### 4. Data Freshness Guarantees

Clear freshness guarantees that meet or exceed current service expectations. No regression in data freshness.

### 5. gRPC API

The API must be exposed via **gRPC** for internal service communication. This provides better performance, type safety, and streaming capabilities. The gRPC should be AIP compliant ([https://google.aip.dev/general](https://google.aip.dev/general)).

### 6. Query Patterns Support

The API must support:

- **Single device queries**: Get state for a specific robot by ID
- **Bulk queries**: Get state for multiple robots efficiently
- **Filtered queries**: Query robots by criteria (e.g., available robots, healthy robots, robots in a location)

### 7. MDS Data Support

The API must provide all data needed by MDS endpoints with appropriate freshness constraints. The MDS Service will query the Device API to obtain this data and transform it to MDS format. The API must include fields necessary for MDS compliance (e.g., vehicle state, event information) and ensure data freshness meets MDS requirements.