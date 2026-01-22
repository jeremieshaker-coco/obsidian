> Goal: Implement the new Device service in Go (coco-services) that consumes events from GreenGrass, DriveU, and AWS IoT, publishes normalized device events, and enables phased migration of consumers and APIs. PoC link: [https://github.com/cocorobotics/delivery-platform/pull/6648Design:](https://github.com/cocorobotics/delivery-platform/pull/6648Design:) Device API Redesign [WIP] - Includes good starting points for various protobuf definitions.

**Note:** The plan and the design are **not aligned** and this is intentional. The design lays out all the things we want to change, regardless of the sequencing. This plan tries to be pragmatic about what can be changed without re-writing an unreasonable amount of code.

---

# Summary of Changes Covered

This plan covers:

- Building a new Device to ingest GreenGrass/DriveU/AWS IoT signals and publish normalized `Device.*` events.
- Incrementally migrating consumers and APIs to the new device event and API surface without double-subscribing.
- Simplifying robot state events while keeping `robotStateHistory` as the source of truth in Operations.
- Replacing `Robots.StateChange` with `Operations.DeviceOperationalStateChanged` (reduced payload), derived from `robotStateHistory`.
- Removing dependency on `robotStateHistory` reads where they were used for current-state fields.
- Deprecating Milestone 4 event publishing for `TripEvents.*`, `DeliveryEvents.*`, `FoTaskEvents.*`, and `DeploymentEvents.*` because APIs replace those state effects.

## Decisions / Goals

- Keep `robotStateHistory` as the source of truth for operational state in Operations.
- Replace `Robots.StateChange` with `Operations.DeviceOperationalStateChanged` (payload only: `hasFood`, `operationState`, `needsMaintenance`, `needsPickup`, `needsMovement`, `undergoingMaintenance`).
- Keep `hasFood` as a legacy field in `Operations.DeviceOperationalStateChanged` until device-reported cargo is available.
- Do not publish `Device.Moved`; movement is derived from `Device.StateUpdated` where needed.
- Drop `driveable` entirely.
- Because of the APIs, we will NOT create these Milestone 4 events:
    - TripEvents.*
    - DeliveryEvents.*
    - FoTaskEvents.*
    - DeploymentEvents.*
- Therefore Milestone 4 event publishing is deprecated/unneeded.

# Outcome

By the end of this implementation plan, we will have:

- A fully implemented Device Service (not including any fleet-related functionality)
- Decommissioned the legacy Device Service
- Restricted delivery-platform dependencies to rely on Device.* events plus Operations-owned robot state (`robotStateHistory`)
- Removed our dependency on OperationState (aside from publishing it to the device shadow)
- Replaced `Robots.StateChange` with `Operations.DeviceOperationalStateChanged`, and removed `IoT.Heartbeat` after migration

---

# Methodology

This plan follows an incremental approach:

1. **Publish first** - Start publishing new events alongside old ones
2. **Migrate consumers** - Update consumers one-by-one to use new events
3. **Migrate APIs** - Move device-related endpoints to the new service
4. **Clean up** - Remove deprecated code after full migration

Each task should be:

- Small and focused (single PR per task where possible)
- Trackable via JIRA ticket
- Include TODO comments when we begin publishing a new event that overlaps an existing one (publisher cleanup tracking)
- Include a safety gate before switching an event source in production (shadow/metrics or feature flag)

Migration stance:

- Double publish when needed, but do not double subscribe
- Each consumer migration swaps the subscription in a single PR (old handler removed or unsubscribed)
- Prefer shadow validation before switching event sources (new handler emits metrics/logs only)

Naming conventions:

- Exchange names use slash-delimited paths (e.g., `device/state-updated`, `device/driveu/connectivity-changed`)
- Routing keys use dot notation when structured; for DeviceEvents we keep routing keys as `{serial}` only

Why publish new events even when similar events exist:

- New events are action-oriented and carry intent/timestamps (vs. derived or repository-level state)
- Routing is serial-based for efficient fan-out to device-scoped consumers
- Payloads include new fields when available (e.g., startedAt/resumedAt/completedAt, deployment location context)
- Overlap is temporary to enable incremental migration without breaking downstream systems

### Device.StateUpdated (Temporary Stream)

`Device.StateUpdated` is a temporary, event-based stand-in for a future gRPC streaming API.
It emits a **full device snapshot** and can be triggered by multiple upstream sources (IoT heartbeat, DriveU connectivity updates, or other state changes).
The single-stream approach intentionally bundles health, connectivity, location, and other fields to reduce race conditions across multiple event streams.

---

# Milestones

## Milestone 1: Device Service Foundation

**Goal**: Set up the Device service within the existing TypeScript service so we can publish normalized events quickly.

### ~~1.1 Service Scaffolding (Go, deprecated)~~

~~- [ ] **TASK-001**: Create new `device` service in `coco-services` repo~~
~~    - Directory structure: `device/`~~
~~    - Basic Go service setup (main, config, healthcheck)~~
~~    - Dockerfile and k8s manifests~~
~~    - CI/CD pipeline configuration~~
~~- [ ] **TASK-002**: Set up RabbitMQ infrastructure~~
~~    - Define new exchanges in `device.events` namespace~~
~~    - Configure DLQ and retry policies~~
~~    - Set up bindings for fan-out to consumers~~
~~- [ ] **TASK-003**: Set up AWS IoT SQS consumer~~
~~    - Configure SQS queue for IoT heartbeat messages~~
~~    - Implement basic message polling and acknowledgment~~
~~    - Add metrics and logging~~
~~- [ ] **TASK-004**: Set up DriveU webhook handler~~
~~    - HTTP endpoint for DriveU status callbacks~~
~~    - Authentication/validation middleware~~
~~    - Event normalization layer~~

### 1.1 Device Service (TypeScript) Scaffolding

- [ ] **TASK-001A**: Add AWS IoT SQS consumer in TypeScript
    - Configure SQS queue for IoT heartbeat messages (used to emit `Device.StateUpdated`)
    - Implement basic message polling and acknowledgment
    - Add metrics and logging
- [ ] **TASK-001B**: Add DriveU webhook handler in TypeScript
    - HTTP endpoint for DriveU status callbacks
    - Authentication/validation middleware
    - Event normalization layer

### 1.2 Data Layer

- [ ] **TASK-005**: Define protobuf schemas for Device.* events
    - `Device.StateUpdated` (full snapshot; temporary stand-in for gRPC stream)
    - `Device.LidOpened` / `Device.LidClosed` / `Device.LidJammed`
    - `Device.BatteryLow` / `Device.BatteryCritical`
    - `Device.ConnectivityChanged`
    - `Device.HealthDegraded` / `Device.HealthRestored`
    - `Device.EmergencyStop`
    - `Device.PinEntry`
    - `Device.CargoChanged` (deferred; requires device-reported cargo)
- [ ] **TASK-006**: Set up device state storage
    - Redis cache for ephemeral device state
    - Define schema for device snapshots
    - Implement get/set operations

---

## Milestone 2: Event Publishing - Device StateUpdated & Health

**Goal**: Start publishing `Device.StateUpdated` (full snapshot) and health-related events from the new service.

### 2.1 Device.StateUpdated

- [ ] **TASK-007**: Implement state update parsing
    - Parse IoT heartbeat payload from SQS
    - Normalize to internal `Device` model
    - Handle schema versioning
- [ ] **TASK-008**: Publish `Device.StateUpdated` event
    - Map internal model to protobuf
    - Publish to `device.events` exchange
    - Routing key: `{serial}`
    - Include `updated_fields` mask and `trigger_source` (heartbeat, DriveU, connectivity)
    - Publish on any upstream change that affects device state, including:
      - IoT heartbeat received
      - DriveU connectivity callback/status change
      - Connectivity state transitions (IoT/DriveU online/offline)
      - Component health changes if tracked outside heartbeat
    - Payload is a full device snapshot (location, battery, connectivity, component health, lid state, etc.)

### 2.2 Device Health Events

- [ ] **TASK-010**: Implement component health tracking
    - Track previous component states per device
    - Detect state transitions (OK → FAULT, FAULT → OK)
    - Debounce rapid state changes (configurable threshold)
- [ ] **TASK-011**: Publish `Device.HealthDegraded` event
    - Trigger on component status degradation
    - Include `component_name`, `status_code`, `fault_code`, `message`
    - Routing key: `{serial}`
- [ ] **TASK-012**: Publish `Device.HealthRestored` event
    - Trigger on component recovery
    - Include `downtime_seconds`
    - Routing key: `{serial}`

---

## Milestone 3: Event Publishing - Connectivity & Lid Events

**Goal**: Publish connectivity and lid-related events from the new service.

### 3.1 Connectivity Events (Separate Streams)

- [ ] **TASK-014**: Implement IoT connectivity tracking (separate stream)
    - Detect online/offline transitions from IoT heartbeats
    - Publish `Device.IotConnectivityChanged`
    - Routing key: `{serial}`
    - Add overlap TODOs in `payload.types.ts` and `names.ts` documenting deprecation of legacy shared connectivity
    - Permalink (`payload.types.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L703](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L703)
    - Permalink (`names.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L137](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L137)
- [ ] **TASK-015**: Implement DriveU connectivity tracking (separate stream)
    - Detect online/offline transitions from DriveU webhooks
    - Publish `Device.DriveuConnectivityChanged`
    - Routing key: `{serial}`
    - Add overlap TODOs in `payload.types.ts` and `names.ts` documenting deprecation of legacy shared connectivity
    - Permalink (`payload.types.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L703](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L703)
    - Permalink (`names.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L137](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L137)
- [ ] **TASK-016**: Update downstream consumers to subscribe to the correct connectivity stream
    - Ensure handlers use IoT vs DriveU streams explicitly (no shared “connectivity” event)
    - Swap subscriptions in a single PR (no double subscribe)
    - Add TODOs in legacy connectivity publisher(s) for cleanup

### 3.2 Lid Events

- [ ] **TASK-017**: Parse lid state from heartbeat
    - Detect lid open/close transitions
    - Track lid open trigger (button, PIN, command, manual)
- [ ] **TASK-018**: Publish `Device.LidOpened` event
    - Include `trigger`, `request_id` (if command-triggered)
    - Routing key: `{serial}`
- [ ] **TASK-019**: Publish `Device.LidClosed` event
    - Include `seconds_open`, `was_timeout`
    - Routing key: `{serial}`
- [ ] **TASK-020**: Publish `Device.LidJammed` event
    - Trigger on lid mechanism fault
    - Include `fault_code`, `message`
    - Routing key: `{serial}`

### 3.3 Battery Events

- [ ] **TASK-021**: Implement battery threshold tracking
    - Track battery level per device
    - Detect downward threshold crossings (20%, 5%)
    - Note: This mirrors current system behavior; future intent is to emit battery range events once range semantics are defined
- [ ] **TASK-022**: Publish `Device.BatteryLow` event
    - Trigger at 20% threshold
    - Routing key: `{serial}`
    - Note: This is a stopgap; prefer range-based signals when available
- [ ] **TASK-023**: Publish `Device.BatteryCritical` event
    - Trigger at 5% threshold
    - Routing key: `{serial}`
    - Note: This is a stopgap; prefer range-based signals when available

### 3.4 Pin Entry, Cargo & Emergency Stop Events

- [ ] **TASK-024**: Publish `Device.PinEntry` event
    - Parse PIN entry from IoT messages
    - Routing key: `{serial}`
- [ ] **TASK-025**: Defer `Device.CargoChanged` (requires device-reported cargo)
    - Keep `hasFood` as the legacy cargo signal in `Operations.DeviceOperationalStateChanged` for this migration
- [ ] **TASK-026**: Publish `Device.EmergencyStop` event
    - Parse emergency stop from IoT messages
    - Include `source` (button, remote, system)
    - Routing key: `{serial}`

---

## Milestone 4: Robot State Events (History-Based)

**Goal**: Keep `robotStateHistory` as the source of truth in Operations while simplifying the robot state event payload.

### 4.1 Operations.DeviceOperationalStateChanged (Replacement for Robots.StateChange)

- [ ] **TASK-025L0**: Publish `Operations.DeviceOperationalStateChanged` from Operations
    - _Definition of Done:_ Event is emitted based on `robotStateHistory` updates (or state transition events) in Operations.
    - Payload fields: `hasFood` (legacy), `operationState`, `needsMaintenance`, `needsPickup`, `needsMovement`, `undergoingMaintenance`
    - Routing key: `{serial}`
    - Double-publish with `Robots.StateChange` during migration; add TODOs for cleanup
    - Publish from the same Operations call sites that currently emit `Robots.StateChange`:
      - `robot-fleet-management.service.ts` (deploy/undeploy): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts
      - `fo-requests.service.ts` (FO request creation): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-requests/fo-requests.service.ts
      - `fo-tasks.service.ts` (maintenance flags): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/fo-tasks.service.ts
      - `delivery.handlers.ts` (delivery termination): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts
      - `pilot-trips.publisher.ts` (trip completion/cancel): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts
      - `pilot-trips.repository.ts` (trip resumption): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts
      - `pilot-trips.service.ts` (trip start): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/services/pilot-trips.service.ts
      - `robots.service.ts` (grounding): https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/services/robots.service.ts
    - _Verification:_ Dispatch Engine and Operations handlers can consume the new payload with parity metrics

### ~~4.2 Deprecated: Service-Level State Events (Trip/Deployment/FoTask/Delivery)~~

~~Trip/Delivery/FoTask/Deployment events are no longer required because the simplified state-change event remains the primary signal. The tasks below are deprecated/unneeded.~~

- [ ] **~~TASK-025A**: Publish `TripEvents.Started` from Operations trip start flows~~ (deprecated)
- [ ] **~~TASK-025B**: Publish `TripEvents.Resumed` from Operations trip resume flows~~ (deprecated)
- [ ] **~~TASK-025C**: Publish `TripEvents.Cancelled` from Operations trip cancel flows~~ (deprecated)
- [ ] **~~TASK-025D**: Publish `TripEvents.Completed` from Operations trip complete flows~~ (deprecated)
- [ ] **~~TASK-025E**: Publish `DeploymentEvents.Deployed` and `DeploymentEvents.Undeployed`~~ (deprecated)
- [ ] **~~TASK-025F**: Publish `FoTaskEvents.Created` and `FoTaskEvents.Updated`~~ (deprecated)
- [ ] **~~TASK-025G**: Publish `DeliveryEvents.Loaded` and `DeliveryEvents.Unloaded`~~ (deprecated)

---

## Milestone 5: State Transition Ownership Shift

**Goal**: Keep State Service as the transition engine initially, but move state-change publishing to Operations and prepare for State decommissioning.

### 5.1 Stop State Service from Publishing Robots.StateChange

- [ ] **TASK-025M1**: Disable `Robots.StateChange` publishing in State Service
    - Update `state-transition.subscriber.ts` to stop publishing `Robots.StateChange` (keep `State.TransitionCompleted`/`State.TransitionFailed`)
    - Ensure `robotStateHistory` writes remain intact during this phase
    - _Verification:_ State transitions still occur; no `Robots.StateChange` emitted from State

### 5.2 Operations Publishes DeviceOperationalStateChanged

- [ ] **TASK-025M2**: Operations consumes `State.TransitionCompleted` and publishes `Operations.DeviceOperationalStateChanged`
    - Map the reduced payload from transition data + `robotStateHistory`
    - Continue double-publish with `Robots.StateChange` until consumers migrate
    - _Verification:_ Parity metrics match legacy `Robots.StateChange`

### 5.3 Prepare for State Service Deletion

- [ ] **TASK-025M3**: Move or duplicate `robotStateHistory` writes into Operations
    - Keep State Service writing history until Operations parity is verified
    - _Verification:_ History records match between State and Operations in shadow mode
- [ ] **TASK-025M4**: Add TODOs and tracking for removing State transition ownership
    - Track remaining dependencies on `/state/:serial/request-transition` and state machine logic

---

## Milestone 6: Consumer Migration - Trip Monitor

**Goal**: Migrate Trip Monitor to consume new Device.StateUpdated event.

### 6.1 Trip Monitor Updates

- [ ] **TASK-026**: Replace IoT.Heartbeat subscription with `Device.StateUpdated`
    - Replace handler in `trip-monitor.handler.ts` (IoT.Heartbeat → Device.StateUpdated)
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts#L19-L23](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts#L19-L23)
    - Map `Device.StateUpdated` to existing `updateFromHeartbeat` interface
    - Add `updateFromDeviceHeartbeat` method (from PoC diff)
    - Movement detection is derived from `Device.StateUpdated`
    - Remove/unsubscribe the old IoT.Heartbeat handler in the same PR (no double subscribe)
- [ ] **TASK-027**: Add operation state derivation to Trip Monitor
    - Track deployment/trip state per serial
    - Implement `deriveOperationState` helper (from PoC diff)
    - Use `Operations.DeviceOperationalStateChanged` (or `robotStateHistory`) instead of Trip/Deployment events
- [ ] **TASK-028**: Add TODO comment in IoT.Heartbeat publisher for deprecation cleanup
    - Add: `// TODO(JIRA-XXX): Remove IoT.Heartbeat publish after Device.StateUpdated migration verified`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/iot-streamer/iot-streamer.service.ts#L574-L628](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/iot-streamer/iot-streamer.service.ts#L574-L628)

## Milestone 7: Consumer Migration - Dispatch Engine

**Goal**: Migrate Dispatch Engine to consume new events instead of `Robots.StateChange`.

### 7.0 Telemetry Migration (Device.StateUpdated)

- [ ] **TASK-028A**: Dispatch Engine telemetry → Device.StateUpdated
    - Replace `IoT.Heartbeat` subscription in `iot-heartbeat.handler.ts` with `Device.StateUpdated`
    - Ensure snapshot includes battery, health, connectivity, and location
    - _Definition of Done:_ Dispatch Engine uses `Device.StateUpdated` only; no IoT.Heartbeat dependency remains
    - _Verification:_ Metrics parity in shadow mode; manual validation that supply cache reflects battery/health/connectivity

### 7.1 Operations.DeviceOperationalStateChanged

- [ ] **TASK-029**: Replace `Robots.StateChange` subscription with `Operations.DeviceOperationalStateChanged`
    - Replace handler in `robot-stage-change.handler.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts#L1-L217](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts#L1-L217)
    - Map fields: `operationState`, `needsMovement`, `needsMaintenance`, `needsPickup`, `undergoingMaintenance`, `hasFood`
    - Do not expect `tripType` or `driveable` in the new payload; derive from `robotStateHistory` if needed
    - Safety: run in shadow mode first (metrics/logging only), compare counts vs legacy handler before switch
- [ ] **TASK-030**: Add TODO comment in `Robots.StateChange` publisher(s) for cleanup
    - Add: `// TODO(JIRA-XXX): Remove Robots.StateChange publish after all Dispatch Engine consumers migrated`

---

## Milestone 8: Consumer Migration - Operations Service

**Goal**: Migrate Operations Service handlers to consume new events.

### 8.0 Telemetry Migration (Device.StateUpdated)

- [ ] **TASK-028B**: Operations telemetry → Device.StateUpdated
    - Replace `IoT.Heartbeat` subscription in `iot-heartbeat-handler.ts` with `Device.StateUpdated`
    - Update cache hydration for health/location/battery/connectivity from Device payloads
    - _Definition of Done:_ Operations uses `Device.StateUpdated` for telemetry cache; no IoT.Heartbeat dependency remains
    - _Verification:_ Manual validation of Operations robot cache values in dev/staging

### 8.1 Operations.DeviceOperationalStateChanged

- [ ] **TASK-041**: Replace `Robots.StateChange` subscription with `Operations.DeviceOperationalStateChanged`
    - Replace handler in `robot-state-change-handler.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts#L52-L68](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts#L52-L68)
    - Map fields: `operationState`, `needsMovement`, `needsMaintenance`, `needsPickup`, `undergoingMaintenance`, `hasFood`
    - Do not expect `tripType` or `driveable` in the new payload; use `robotStateHistory` where needed
    - Remove/unsubscribe the old `Robots.StateChange` handler in the same PR
- [ ] **TASK-042**: Add TODO comment in `Robots.StateChange` publisher(s) for cleanup
    - Add: `// TODO(JIRA-XXX): Remove Robots.StateChange publish after Operations consumers migrated`

---

## Milestone 9: Consumer Migration - State Service (→ Lid Service)

**Goal**: Migrate State Service off `Robots.StateChange`; use `Operations.DeviceOperationalStateChanged` only if maintenance flags are still required.

### 9.1 Operations.DeviceOperationalStateChanged (If Still Needed)

- [ ] **TASK-046**: Replace `Robots.StateChange` subscription with `Operations.DeviceOperationalStateChanged`
    - Replace handler in `robot-state.handler.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/robot-state.handler.ts#L18-L29](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/robot-state.handler.ts#L18-L29)
    - Update `needsMaintenance` shadow from the reduced payload; do not rely on event payloads for other fields
    - Remove/unsubscribe the old `Robots.StateChange` handler in the same PR
- [ ] **TASK-047**: Drop direct event-driven maintenance updates if they are no longer required
    - Same handler file/permalink as above
    - If maintenance is derived elsewhere, remove event handlers entirely
- [ ] **TASK-048**: Add TODO comment in `Robots.StateChange` publisher(s) for cleanup
    - Add: `// TODO(JIRA-XXX): Remove Robots.StateChange publish after State Service consumers migrated`

---

---

## Milestone 10: Publisher Migration - Remove Robots.StateChange Publishers

**Goal**: Remove `publishRobotStateUpdated` calls from all services now that consumers use `Operations.DeviceOperationalStateChanged`.

### 10.1 Operations Service Cleanup

- [ ] **TASK-049**: Remove `publishRobotStateUpdated` from `robot-fleet-management.service.ts`
    - Undeployment no longer needs to publish state change
    - Deployment no longer needs to publish state change
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fleet-management/services/robot-fleet-management.service.ts)
- [ ] **TASK-050**: Remove `publishRobotStateUpdated` from `fo-requests.service.ts`
    - FO request creation no longer needs to publish state change
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-requests/fo-requests.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-requests/fo-requests.service.ts)
- [ ] **TASK-051**: Remove `publishRobotStateUpdated` from `fo-tasks.service.ts`
    - Task creation/completion no longer needs to publish state change
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/fo-tasks.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/fo-tasks.service.ts)
- [ ] **TASK-052**: Remove `publishRobotStateUpdated` from `delivery.handlers.ts`
    - Delivery termination no longer needs to publish state change
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/fo-tasks/handlers/delivery.handlers.ts)
- [ ] **TASK-053**: Remove `publishRobotStateUpdated` from `pilot-trips.publisher.ts`
    - Trip completion/cancellation no longer rely on robot state events; use `robotStateHistory`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/publishers/pilot-trips.publisher.ts)
- [ ] **TASK-054**: Remove `publishRobotStateUpdated` from `pilot-trips.repository.ts`
    - Trip resumption no longer relies on robot state events; use `robotStateHistory`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/repositories/pilot-trips.repository.ts)
- [ ] **TASK-055**: Remove `publishRobotStateUpdated` from `pilot-trips.service.ts`
    - Trip start no longer relies on robot state events; use `robotStateHistory`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/services/pilot-trips.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/trips/services/pilot-trips.service.ts)
- [ ] **TASK-056**: Remove `publishRobotStateUpdated` from `robots.service.ts`
    - Grounding robot no longer needs separate state change publish
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/services/robots.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/services/robots.service.ts)

### 10.2 Deliveries Service Cleanup

- [ ] **TASK-057**: Remove `publishRobotStateUpdated` from `robot.service.ts`
    - Loading robot no longer relies on robot state events; use Deliveries.AttemptTransitioned/ProviderUpdated + APIs
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/providers/robot/robot.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/providers/robot/robot.service.ts)
- [ ] **TASK-058**: Remove `publishRobotStateUpdated` from `delivery.service.ts`
    - Rescue attempt failure no longer relies on robot state events; use Deliveries.AttemptTransitioned/ProviderUpdated + APIs
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/delivery/service/delivery.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/delivery/service/delivery.service.ts)
- [ ] **TASK-059**: Remove `publishRobotStateUpdated` method from `amqp-publisher.service.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/shared/amqp-publisher.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/shared/amqp-publisher.service.ts)

### 10.3 State Service Cleanup

- [ ] **TASK-060**: Remove `publishRobotStateUpdated` from `state-transition.subscriber.ts`
    - State transitions no longer need to publish Robots.StateChange
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state-machine/listeners/state-transition.subscriber.ts)
- [ ] **TASK-061**: Remove `publishRobotStateUpdated` method from `publisher.service.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/publisher/services/publisher.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/publisher/services/publisher.service.ts)
- [ ] **TASK-062**: Remove `PublisherService` dependency from `state-transition.service.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state-transition.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state-transition.service.ts)

### 10.4 Operations Service Publisher Cleanup

- [ ] **TASK-063**: Remove `publishRobotStateUpdated` method from Operations `publisher.service.ts`
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/publisher/services/publisher.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/publisher/services/publisher.service.ts)

---

## Milestone 11: API Migration - Device Read gRPC Endpoints

**Goal**: Move device-related read endpoints to gRPC in the new Device service.

### 11.1 Device Info Endpoints (gRPC)

- [ ] **TASK-064**: Implement `ListDevices` gRPC in Device
    - List devices with filtering
    - Data from device registry + cache
- [ ] **TASK-065**: Implement `GetDevice` gRPC in Device
    - Single device details
    - Combine registry + latest state
- [ ] **TASK-066**: Implement `BatchGetDeviceGps` gRPC in Device
    - Batch GPS locations from cache
- [ ] **TASK-067**: Implement `GetLatestDeviceStates` gRPC in Device
    - Latest device states from cache

### 11.2 Heartbeat History Endpoints (gRPC)

- [ ] **TASK-068**: Implement `ListDeviceHeartbeatHistory` gRPC in Device
    - Query historical heartbeat data
    - Support time range filtering
- [ ] **TASK-069**: Implement `GetLatestDeviceHeartbeat` gRPC in Device
    - Latest heartbeat per device

### 11.3 Command History Endpoints (gRPC)

- [ ] **TASK-070**: Implement `ListDeviceCommandResponses` gRPC in Device
    - Recent command responses
- [ ] **TASK-071**: Implement `ListDeviceCommandRequests` gRPC in Device
    - Recent command requests
- [ ] **TASK-072**: Implement `GetDeviceCommandResponse` gRPC in Device
    - Lookup by request ID

---

## Milestone 12: API Migration - State Service gRPC Endpoints

**Goal**: Move state-related read endpoints to Device gRPC.

### 12.1 State Endpoints (gRPC)

- [ ] **TASK-073**: Implement `GetRobotOperationalState` gRPC in Device
    - Current robot operational state
    - Derive from deployment + trip + issues
- [ ] **TASK-074**: Implement `ListRobotStateHistory` gRPC in Device
    - State history query
    - Migrate from State Service

### 12.2 Connectivity Endpoints (gRPC)

- [ ] **TASK-075**: Implement `GetRobotConnectivityOverall` gRPC in Device
    - Aggregated connectivity status
    - From DriveU + IoT sources
- [ ] **TASK-076**: Implement `ListRobotConnectivityHistory` gRPC in Device
    - Connectivity history query

---

## Milestone 13: API Migration - Device Command gRPC Endpoints

**Goal**: Move device command endpoints to the new Device gRPC service.

### 13.1 Lid Commands (gRPC)

- [ ] **TASK-077**: Implement `SendLidCommand` gRPC in Device
    - Direct lid command to device
    - Write to IoT shadow
- [ ] **TASK-078**: Implement `SendLidCommandWithContext` gRPC in Device
    - Synchronous lid command with state context
    - Support PIN flow integration

### 13.2 Reset Commands (gRPC)

- [ ] **TASK-079**: Implement `SendSoftReset` gRPC in Device
- [ ] **TASK-080**: Implement `SendHardReset` gRPC in Device
- [ ] **TASK-081**: Implement `SendModemReset` gRPC in Device
- [ ] **TASK-082**: Implement `SendSwitchSimBackup` gRPC in Device
- [ ] **TASK-083**: Implement `SendResetGps` gRPC in Device

### 13.3 Other Commands (gRPC)

- [ ] **TASK-084**: Implement `SendLightsCommand` gRPC in Device
- [ ] **TASK-085**: Implement `SignalLoadFailure` gRPC in Device
- [ ] **TASK-086**: Implement `SendDoordashVerifyFailure` gRPC in Device
- [ ] **TASK-087**: Implement `StartWebRtcAuth` gRPC in Device

---

## Milestone 14: API Migration - State Service Write gRPC Endpoints

**Goal**: Move state transition and PIN-related endpoints to Device gRPC.

### 14.1 State Transition (gRPC)

- [ ] **TASK-088**: Implement `RequestRobotStateTransition` gRPC in Device
    - This becomes issue-based in new architecture
    - Map old transitions to issue creation/clearing
    - GROUNDED → create MANUAL_GROUND issue
    - Clearing GROUNDED → clear issues

### 14.2 PIN & Route Endpoints (gRPC)

- [ ] **TASK-089**: Implement `SetUnlockPin` gRPC in Device
    - Temporary PIN for pickup flows
- [ ] **TASK-090**: Implement `RemoveUnlockPin` gRPC in Device
    - Clear temporary PIN
- [ ] **TASK-091**: Implement `SetRouteId` gRPC in Device
    - Associate robot with route

---

## Milestone 15: Client Migration

**Goal**: Update all clients to use new Device gRPC endpoints (via internal client or API).

### 15.1 MRO Web

- [ ] **TASK-092**: Update MRO to use Device for device endpoints
    - `useRobotsList.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotsList.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotsList.ts))
    - `useRobotDetails.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotDetails.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotDetails.ts))
    - `useRobotGeolocation.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotGeolocation.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotGeolocation.ts))
    - `useRobotState.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotState.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotState.ts))
    - `useRobotStateHistory.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotStateHistory/useRobotStateHistory.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotStateHistory/useRobotStateHistory.ts))
    - `useRobotConnectivity.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotConnectivity.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotConnectivity.ts))
    - `useRobotConnectivityHistory.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotConnectivityHistory/useRobotConnectivityHistory.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useRobotConnectivityHistory/useRobotConnectivityHistory.ts))
    - `useRobotHeartbeat.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotHeartbeat.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotHeartbeat.ts))
    - `useRobotHealth.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotHealth.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotHealth.ts))
    - `useRobotRawHeartbeat.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotRawHeartbeat.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/useRobotRawHeartbeat.ts))
    - `useCommandsResponses.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useCommandsResponses.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useCommandsResponses.ts))
    - `useCommandsRequests.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useCommandsRequests.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useCommandsRequests.ts))
    - `useSendLidCommand.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useSendLidCommand.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/commands/useSendLidCommand.ts))
    - `useChangeStateStatus.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useChangeStateStatus.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mro/src/features/device/modules/robots/api/state/useChangeStateStatus.ts))

### 15.2 Mission Control

- [ ] **TASK-093**: Update Mission Control to use Device
    - `device.api.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/api/device.api.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/api/device.api.ts))
    - `robot-state.api.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/api/robot-state.api.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/api/robot-state.api.ts))
    - `useRobotDetails.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/useRobotDetails.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/useRobotDetails.ts))
    - `useRobotLatestStates.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/useRobotLatestStates.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/useRobotLatestStates.ts))
    - `useCommandsResponses.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/commands/useCommandsResponses.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/commands/useCommandsResponses.ts))
    - `useCommandsRequests.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/commands/useCommandsRequests.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/mission-control/src/features/device/api/robots/commands/useCommandsRequests.ts))

### 15.3 Field Ops App

- [ ] **TASK-094**: Update Field Ops app to use Device
    - `useReportBotIssue.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useReportBotIssue.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useReportBotIssue.ts))
    - `useUpdateBotLid.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useUpdateBotLid.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useUpdateBotLid.ts))
    - `useUpdateBotLights.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useUpdateBotLights.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/mutations/useUpdateBotLights.ts))
    - `useRobotLocation.ts` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/queries/useRobotLocation.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/client/field-ops/src/hooks/queries/useRobotLocation.ts))

### 15.4 Pilot Web

- [ ] **TASK-095**: Update Pilot web to use Device
    - `useSendSoftReset.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendSoftReset.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendSoftReset.tsx))
    - `useSendModemReset.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendModemReset.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendModemReset.tsx))
    - `useSendHardReset.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendHardReset.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendHardReset.tsx))
    - `useSendSwitchSimBackup.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendSwitchSimBackup.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendSwitchSimBackup.tsx))
    - `useSendResetGPS.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendResetGPS.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/hooks/useSendResetGPS.tsx))
    - `route.ts` (set-route-id) (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/experimental/lib/route.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/pilot/src/experimental/lib/route.ts))

### 15.5 Merchant Fleet

- [ ] **TASK-096**: Update Merchant Fleet to use Device
    - `useToggleRobotLid.tsx` (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/merchant-fleet/src/hooks/useToggleRobotLid.tsx](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/web/merchant-fleet/src/hooks/useToggleRobotLid.tsx))

### 15.6 Service-to-Service Clients

- [ ] **TASK-097**: Update State Service to call Device for lid commands
    - `state.service.ts` → Device instead of direct Device Service (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state.service.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/state.service.ts))
- [ ] **TASK-098**: Update Deliveries Service deviceless client
    - `deviceless.client.ts` → Device (Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/deviceless/deviceless.client.ts](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/deliveries/src/modules/deviceless/deviceless.client.ts))

---

## Milestone 16: Final Cleanup

**Goal**: Remove deprecated code, exchanges, and old services.

### 16.1 Remove Old Event Handlers

- [ ] **TASK-099**: Remove `Robots.StateChange` handler from Dispatch Engine
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts#L1-L217](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/dispatch-engine/src/modules/eventsink/handlers/robot-stage-change.handler.ts#L1-L217)
- [ ] **TASK-100**: Remove `Robots.StateChange` handler from Operations Service
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts#L52-L198](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts#L52-L198)
- [ ] **TASK-101**: Remove `Robots.StateChange` handler from State Service
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/robot-state.handler.ts#L17-L52](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/state/src/state/robot-state.handler.ts#L17-L52)
- [ ] **TASK-102**: Remove `IoT.Heartbeat` handler from Trip Monitor (after Device.StateUpdated verified)
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts#L19-L33](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/service/trip-monitor/src/modules/trip-monitor/trip-monitor.handler.ts#L19-L33)

### 16.2 Remove Old Exchanges

- [ ] **TASK-103**: Deprecate `Robots.StateChange` exchange
    - Mark as deprecated in `names.ts`
    - Schedule removal after monitoring period
    - Permalink: [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L139-L148](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L139-L148)
- [ ] **TASK-104**: Remove `Robots.StateChange` from exchange definitions
    - After all consumers migrated
    - Permalinks:
        - [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L139-L148](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L139-L148)
        - [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L706-L712](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L706-L712)
        - [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/routing-keys.builders.ts#L192-L192](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/routing-keys.builders.ts#L192-L192)
        - [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/routing-keys.types.ts#L219-L219](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/routing-keys.types.ts#L219-L219)

### 16.3 Service Cleanup

- [ ] **TASK-105**: Remove telemetry processing from State Service
    - Move becomes Lid Service only
- [ ] **TASK-106**: Remove connectivity handling from State Service
    - Moved to Device
- [ ] **TASK-107**: Decommission the legacy Device Service
    - After Device has full coverage, and all consumers are updated, we should be able to turn down the legacy device service, and delete its code

### 16.4 Documentation

- [ ] **TASK-109**: Update architecture documentation
    - Document Device service
    - Update event flow diagrams
    - Update API documentation
- [ ] **TASK-110**: Remove all cleanup TODO comments
    - Search for `TODO(JIRA-XXX)` comments added during migration
    - Verify each is addressed
    - Remove comments

---

## Tracking & Cleanup Strategy

### TODO Comment Convention

During migration, add TODO comments in this format:

```tsx
// TODO(DEVICE-XXX): <description of cleanup needed>
// Remove after: <condition>

```

Preferred placement:

- In the legacy publisher for any event that now has a new overlapping event stream
- In `payload.types.ts` and `names.ts` where overlap exists (annotate intent and deprecation plan)
    - Permalink (`payload.types.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L712](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/payload.types.ts#L662-L712)
    - Permalink (`names.ts`): [https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L148](https://github.com/cocorobotics/delivery-platform/blob/32d08502ce026882ba65ce22062e5704399d0337/lib/common/src/exchanges/names.ts#L95-L148)

### Cleanup Ticket Checklist

Create a single JIRA epic for cleanup tracking. Each cleanup task links to:

1. The file/function to be removed
2. The replacement (new handler/endpoint)
3. Verification criteria before removal

### Verification Criteria

Before removing old code:

1. New handlers receiving events (metrics)
2. No errors in new handlers for 7 days
3. Feature parity verified in staging
4. Old handler receives no events (or only duplicates)

---

## Risk Mitigation

### Dual Publishing

During migration, both old and new events will be published:

- Device publishes new events
- State Service continues publishing old events
- Consumers gradually migrate to new events
- Old publishing removed only after all consumers migrated

### Feature Flags

Consider feature flags for:

- `device.publish-events`: Enable new event publishing
- `device.api-enabled`: Enable new API endpoints
- Per-consumer flags to switch event source

### Rollback Plan

1. If new event handlers fail, old handlers remain as fallback
2. API endpoints can be reverted via routing config
3. Old exchanges/queues retained during migration period

---

## Milestone Dependencies

```
M1 (Foundation) ─────────────────────────────────────────────────────────────────┐
                                                                                 │
M2 (Heartbeat/Health) ──┬─── M6 (Trip Monitor) ──────────────────────────────────┤
                        │                                                        │
M3 (Connectivity/Lid) ──┼────────────────────────────────────────────────────────┤
                        │                                                        │
M4 (Robot State APIs) ──┬── M5 (API Migration) ──┬── M7 (Dispatch Engine) ───┬── M10 (Remove Publishers)─┤
                        │                        │                          │                             │
                        │                        ├── M8 (Operations) ────────┤                             │
                        │                        │                          │                             │
                        │                        └── M9 (State Service) ─────┘                             │
                                                                                 │
M11 (Read APIs) ─────────┬─── M15 (Client Migration) ─── M16 (Final Cleanup) ────┘
                         │
M12 (State APIs) ────────┤
                         │
M13 (Command APIs) ──────┤
                         │
M14 (Write APIs) ────────┘

```