# Device.SnapshotUpdated Implementation Plan

## Overview

This plan covers implementing the `Device.SnapshotUpdated` event, which provides the minimum viable device telemetry needed to replace `IoT.Heartbeat`.

**Key Design Decision:** Based on analysis of actual consumer usage, this event contains ONLY the fields consumers use for decision-making:
- `device_id` - device identifier
- `location` - GPS coordinates with accuracy and heading
- `battery_soc_percent` - aggregate state of charge (just the percentage)
- `healthy` - derived boolean from component health
- `online` - aggregated connectivity from IoT + DriveU

**What this event does NOT include:**
- Operational state (`needsMovement`, `hasFood`, `tripType`, etc.) - remains in `Robots.StateChange`
- Full battery details (voltage, current, temperature) - not used by consumers
- Component health breakdown - consumers only need the derived `healthy` boolean
- DriveU session details (streamer status, session ID, pilot connected) - consumers only need `online`
- Greengrass status - no current consumers

---

## Proto Status

✅ **Completed** - Simplified proto in `coco-protos/coco/dp/device/v1alpha/device_snapshot.proto`:

```protobuf
message DeviceSnapshotUpdated {
  string device_id = 1;              // Device serial, e.g. "C10190"
  Location location = 2;             // GPS: lat, lng, heading, accuracy
  int32 battery_soc_percent = 3;     // Aggregate SOC (0-100)
  bool healthy = 4;                  // Derived from component health
  bool online = 5;                   // Aggregated IoT + DriveU connectivity
  google.protobuf.Timestamp create_time = 100;
}
```

**Files in `coco/dp/device/v1alpha/`:**
| File | Purpose |
|------|---------|
| `device_snapshot.proto` | `DeviceSnapshotUpdated` event (NEW) |
| `device.proto` | Device resource (unchanged) |
| `device_api.proto` | Device API service (unchanged) |
| `location.proto` | Shared `Location` type (unchanged) |
| `battery.proto` | Full `Battery` type for Device API (unchanged) |

**Deleted files** (not needed for minimum viable event):
- `component_health.proto` - consumers only need `healthy` boolean
- `driveu.proto` - consumers only need `online` boolean
- `greengrass.proto` - no current consumers
- `iot_connectivity.proto` - consumers only need `online` boolean
- `lid_state.proto` - no current consumers

---

## Consumer Analysis

### IoT.Heartbeat Consumers - Fields Actually Used

| Consumer | Fields Used | Decision Made |
|----------|-------------|---------------|
| **Dispatch Engine** | `serial`, `battery.soc`, `healthy`, `location.{lat,lng}` | Update robot supply cache for planning |
| **Trip Monitor** | `serial`, `location.{lat,lng,updatedAt}` | Update trip trace, calculate ETA |
| **Operations Service** | `serial`, `healthy`, `battery.soc`, `location.{lat,lng}` | Save heartbeat to DB, cache health |

### Fields NOT Used by Consumers
- `battery.isCharging`, `battery.sn`, voltage, current, temperature
- `location.errHorz` (only logged for diagnostics)
- `location.heading` (only logged for diagnostics)
- `components.*` (consumers just use the derived `healthy`)
- `operationState` (only used to force `healthy=false` when GROUNDED)

### Connectivity Consumers

| Consumer | Event | Field Used |
|----------|-------|------------|
| **Operations Service** | `State.ConnectivityOverallChanged` | `overall.overallStatus === ONLINE` only |

Consumers don't care whether connectivity is from IoT or DriveU - just the aggregate `online` status.

---

## Phase 1: Device Service Infrastructure

### 1.1 Add Proto Generation to Device Service

**Goal:** Generate TypeScript types from the new proto.

**Tasks:**
- [ ] Verify `coco-protos` dependency in Device Service
- [ ] Generate TypeScript interfaces from `device_snapshot.proto`
- [ ] Verify generated types match proto definitions

**Files to verify:**
- `service/device/package.json`

---

### 1.2 Create Snapshot Storage Module

**Goal:** Store the latest snapshot for each device in Redis for aggregation.

**Tasks:**
- [ ] Create `service/device/src/snapshot/` module
- [ ] Implement `SnapshotStorageService` with:
  - `getSnapshot(deviceId: string): DeviceSnapshotUpdated | null`
  - `updateFromHeartbeat(deviceId: string, heartbeat: HeartbeatV1): DeviceSnapshotUpdated`
  - `updateConnectivity(deviceId: string, online: boolean): DeviceSnapshotUpdated`
- [ ] Each update method should:
  1. Get or create snapshot
  2. Update relevant fields
  3. Recompute `healthy` and `online` derived fields
  4. Set `create_time` to now
  5. Return the updated snapshot

**Files to create:**
- `service/device/src/snapshot/snapshot.module.ts`
- `service/device/src/snapshot/snapshot-storage.service.ts`

---

### 1.3 Create Event Publisher Module

**Goal:** Publish `Device.SnapshotUpdated` events to RabbitMQ.

**Tasks:**
- [ ] Add exchange definitions to `lib/common/src/exchanges/`:
  - Add `Device.SnapshotUpdated` to `names.ts`
  - Add payload type to `payload.types.ts`
  - Add routing key builder to `routing-keys.builders.ts`
- [ ] Implement `DeviceEventPublisher` service with:
  - `publishSnapshotUpdated(snapshot: DeviceSnapshotUpdated): Promise<void>`
- [ ] Configure RabbitMQ connection in Device Service

**Files to create:**
- `service/device/src/events/device-events.module.ts`
- `service/device/src/events/device-event-publisher.service.ts`

**Files to modify:**
- `lib/common/src/exchanges/names.ts`
- `lib/common/src/exchanges/payload.types.ts`
- `lib/common/src/exchanges/routing-keys.builders.ts`
- `lib/common/src/exchanges/routing-keys.types.ts`

---

## Phase 2: Ingest IoT Heartbeat

### 2.1 Add SQS Consumer for IoT Heartbeats

**Goal:** Device Service consumes IoT heartbeat messages from SQS.

**Decision Point:**
- **Option A:** Create a new SQS queue for Device Service (recommended - no coupling)
- **Option B:** Share the queue with State Service (requires coordination)

**Tasks:**
- [ ] Configure SQS queue URL in Device Service config
- [ ] Create `service/device/src/iot/` module
- [ ] Implement `IotHeartbeatConsumer` that:
  1. Consumes messages from SQS
  2. Parses the heartbeat payload
  3. Calls `SnapshotStorageService.updateFromHeartbeat()`
  4. Triggers `DeviceEventPublisher.publishSnapshotUpdated()`
- [ ] Add health check for SQS consumer

**Files to create:**
- `service/device/src/iot/iot.module.ts`
- `service/device/src/iot/iot-heartbeat.consumer.ts`

**Infrastructure:**
- [ ] Create/configure SQS queue (if new queue)
- [ ] Update IoT Rule to route to new queue (if new queue)

---

### 2.2 Map Heartbeat to Snapshot

**Goal:** Transform raw IoT heartbeat into `DeviceSnapshotUpdated` fields.

**Mapping:**

| Heartbeat Field | Snapshot Field |
|-----------------|----------------|
| `serial` | `device_id` |
| `battery.soc` | `battery_soc_percent` |
| `location.latitude` | `location.coordinates.latitude` |
| `location.longitude` | `location.coordinates.longitude` |
| `location.heading` | `location.heading_degrees` |
| `location.errHorz` | `location.horizontal_accuracy_meters` |
| `location.updatedAt` | `location.update_time` |
| `healthy` | `healthy` (pass through) |

**Tasks:**
- [ ] Implement mapping logic in `SnapshotStorageService.updateFromHeartbeat()`
- [ ] Add unit tests for mapping

---

## Phase 3: Ingest Connectivity Status

### 3.1 Subscribe to State.ConnectivityOverallChanged

**Goal:** Device Service subscribes to connectivity updates to derive `online` field.

**Rationale:** Consumers only check `overall.overallStatus === ONLINE`. They don't care about IoT vs DriveU breakdown. We can consume the existing aggregated event.

**Tasks:**
- [ ] Create `service/device/src/connectivity/` module
- [ ] Implement `ConnectivityStatusHandler` that:
  1. Subscribes to `State.ConnectivityOverallChanged`
  2. Extracts `overall.overallStatus === Status.ONLINE`
  3. Calls `SnapshotStorageService.updateConnectivity()`
  4. Triggers `DeviceEventPublisher.publishSnapshotUpdated()`

**Files to create:**
- `service/device/src/connectivity/connectivity.module.ts`
- `service/device/src/connectivity/connectivity-status.handler.ts`

---

## Phase 4: Derived Fields

### 4.1 Compute `healthy` Field

**Goal:** Pass through the `healthy` boolean from IoT heartbeat.

The State Service already computes `healthy` from component health in `iot-streamer.service.ts`. We can simply pass this through.

**Logic:**
```typescript
function updateFromHeartbeat(deviceId: string, heartbeat: HeartbeatV1): DeviceSnapshotUpdated {
  return {
    device_id: deviceId,
    // ... location, battery_soc_percent ...
    healthy: heartbeat.healthy,  // Pass through
    online: this.getOnlineStatus(deviceId),
    create_time: new Date(),
  };
}
```

---

### 4.2 Compute `online` Field

**Goal:** Derive `online` from connectivity events.

**Logic:**
```typescript
function updateConnectivity(deviceId: string, online: boolean): DeviceSnapshotUpdated {
  const snapshot = this.getSnapshot(deviceId) ?? this.createEmptySnapshot(deviceId);
  return {
    ...snapshot,
    online: online,
    create_time: new Date(),
  };
}
```

**Tasks:**
- [ ] Store connectivity status per device in Redis
- [ ] Handle cases where heartbeat arrives before connectivity event (default to last known)

---

## Phase 5: Consumer Migration

### 5.1 Consumer Inventory

| Consumer | Current Event | New Event | Fields Needed |
|----------|--------------|-----------|---------------|
| **Dispatch Engine** | `IoT.Heartbeat` | `Device.SnapshotUpdated` | `device_id`, `battery_soc_percent`, `healthy`, `location` |
| **Trip Monitor** | `IoT.Heartbeat` | `Device.SnapshotUpdated` | `device_id`, `location` |
| **Operations Service** | `IoT.Heartbeat`, `State.ConnectivityOverallChanged` | `Device.SnapshotUpdated` | `device_id`, `healthy`, `battery_soc_percent`, `location`, `online` |

**Note:** `Robots.StateChange` is NOT replaced - operational state is separate from device telemetry.

### 5.2 Migration Strategy

1. **Dual-publish:** Device Service publishes `Device.SnapshotUpdated` alongside existing `IoT.Heartbeat`
2. **Shadow mode:** Consumers subscribe to new event, log/compare with old event
3. **Switch:** Consumers switch to new event, unsubscribe from old
4. **Cleanup:** Eventually deprecate `IoT.Heartbeat` (State Service can stop publishing)

---

## Phase 6: Testing & Validation

### 6.1 Unit Tests

- [ ] `SnapshotStorageService.updateFromHeartbeat()` - Test field mapping
- [ ] `SnapshotStorageService.updateConnectivity()` - Test online derivation
- [ ] `DeviceEventPublisher` - Test event publishing

### 6.2 Integration Tests

- [ ] End-to-end: IoT heartbeat SQS → `Device.SnapshotUpdated` RabbitMQ
- [ ] End-to-end: `State.ConnectivityOverallChanged` → `Device.SnapshotUpdated`
- [ ] Aggregation: Heartbeat + connectivity update same snapshot

### 6.3 Validation in Staging

- [ ] Compare `Device.SnapshotUpdated` payload with `IoT.Heartbeat` for same devices
- [ ] Verify all fields present and correct
- [ ] Verify event timing matches expectations
- [ ] Load test with production-like traffic

---

## Work Breakdown Summary

### Milestone 1: Foundation
- [ ] 1.1 Add proto generation
- [ ] 1.2 Create snapshot storage module
- [ ] 1.3 Create event publisher module

### Milestone 2: IoT Integration
- [ ] 2.1 Add SQS consumer
- [ ] 2.2 Implement heartbeat mapping

### Milestone 3: Connectivity Integration
- [ ] 3.1 Subscribe to State.ConnectivityOverallChanged

### Milestone 4: Testing
- [ ] 6.1 Unit tests
- [ ] 6.2 Integration tests

### Milestone 5: Staging Validation & Migration
- [ ] 6.3 Deploy and validate in staging
- [ ] 5.2 Begin consumer migration

---

## Open Questions

1. **SQS Queue Strategy:** Should Device Service have its own IoT heartbeat queue, or share with State Service?
2. **Snapshot TTL:** Should snapshots expire if no updates received? If so, what's the TTL?
3. **Event Rate:** Should we throttle `Device.SnapshotUpdated` events (e.g., max once per 5 seconds per device)?

---

## Related Documents

- [Device API Redesign v3](../designs/Device%20API%20Redesign%20v3.md)
- [Device Events Producers and Consumers](../research/DEVICE_EVENTS.md)
- [Device Gateway Implementation Plan](./device-gateway-implementation-plan.md)
