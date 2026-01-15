## Execution Plan: Device Events Refactoring (POC)

### Goal
Ship a proof-of-concept that starts publishing the new Device events from `Device API Redesign v3` and updates downstream consumers based on the current event inventory in `DEVICE_EVENTS.md`. This plan avoids any Redis or local state store work and focuses only on: event definitions, publishing, and consumer migration.

---

### Scope Boundaries (POC)
- No Redis, no local cache, no derived operational state store.
- We do not remove legacy events yet; we dual-publish during the POC.
- Only migrate the services listed in `DEVICE_EVENTS.md` and the consumers that depend on `operationState`.
- When a legacy event is the only source of a business signal, define a new event for that signal and publish it at the source.

---

### Phase 1: Define New Device Events (Interfaces + Names)

**Goal:** Introduce event interfaces in `delivery-platform/service/device` for each Device event in `Device API Redesign v3`, and make them available to publishers/consumers.

**Deliverables:**
1. Event names and routing keys for Device events (e.g., `Device.Heartbeat`, `Device.Moved`, `Device.HealthDegraded`, `Device.HealthRestored`, etc.).
2. TypeScript interfaces per event in `service/device` (one file per event or grouped by area).
3. Update `lib/common/src/exchanges/names.ts`, `routing-keys.types.ts`, and `payload.types.ts` to include Device events.

**Work items:**
- Create a `service/device/src/events/` folder:
  - `device-heartbeat.event.ts`
  - `device-health.event.ts`
  - `device-connectivity.event.ts`
  - `device-lid.event.ts`
  - `device-battery.event.ts`
  - `device-cargo.event.ts`
  - `device-emergency.event.ts`
- Add corresponding payload mappings to the shared exchange type definitions.

---

### Phase 2: Publish New Device Events (Producers)

**Goal:** Publish the new Device events from the existing telemetry/IoT pipeline without removing legacy events.

**Primary publishers (from `DEVICE_EVENTS.md`):**
- `State Service` publishes:
  - `IoT.Heartbeat`
  - `IoT.HealthStateChanged`
  - `Robots.StateChange`

**POC approach:**
- Keep all existing IoT/Robots events publishing.
- Add Device events alongside the existing ones (dual-publish).

**Publisher mapping (examples):**
- `IoT.Heartbeat` → also publish `Device.Heartbeat`
- `IoT.HealthStateChanged` → publish `Device.HealthDegraded` or `Device.HealthRestored`
- `State.ConnectivityOverallChanged` → publish `Device.ConnectivityChanged`
- `Robots.PinEntry` → publish `Device.PinEntry`
- `Robots.LidOpen` / `Robots.LidClose` → publish `Device.LidOpened` / `Device.LidClosed`

---

### Phase 3: Replace `Robots.StateChange` with Specific Domain Events

**Goal:** Remove reliance on `Robots.StateChange` where it is currently used as a generic event source. For each usage, define a new event that maps to a single domain action and publish it at the source.

#### 3.1 Examples of new events to introduce

| Current Producer | Legacy Event | Replace With | Example Source |
|-----------------|-------------|--------------|----------------|
| Operations Trips | `Robots.StateChange` | `Trip.Started` | `pilot-trips.service.ts` |
| Operations Trips | `Robots.StateChange` | `Trip.Completed` | `pilot-trips.publisher.ts` |
| Operations Trips | `Robots.StateChange` | `Trip.Cancelled` | `pilot-trips.service.ts` |
| Operations Fleet | `Robots.StateChange` | `Deployment.Started` | `robot-fleet-management.service.ts` |
| Operations Fleet | `Robots.StateChange` | `Deployment.Ended` | `robot-fleet-management.service.ts` |
| Operations FO | `Robots.StateChange` | `FoTask.Created` / `FoTask.Updated` | `fo-tasks.service.ts` |
| Deliveries | `Robots.StateChange` | `Delivery.Loaded` / `Delivery.Unloaded` | `robot.service.ts` |

#### 3.2 Concrete example: `pilot-trips.service.ts`

Current behavior (line ~240):
- Publishes `Robots.StateChange` when a trip starts.

New behavior:
- Publish `Trip.Started` with the relevant payload:
  - `serial`
  - `tripId`
  - `tripType`
  - `startedAt`
  - `hasCargo` (if available)

This should be applied consistently to all `Robots.StateChange` producers listed in `DEVICE_EVENTS.md`.

---

### Phase 4: Consumer Migration (Based on `DEVICE_EVENTS.md`)

**Goal:** Update consumers to listen to the new Device/Domain events instead of `operationState` or `Robots.StateChange`.

#### 4.1 Dispatch Engine
- Current dependencies:
  - `IoT.Heartbeat` (uses `operationState`)
  - `Robots.StateChange`
- Update to consume:
  - `Device.Heartbeat` (location, battery, health)
  - `Device.HealthDegraded` / `Device.HealthRestored`
  - `Trip.Started` / `Trip.Completed`
  - `Deployment.Started` / `Deployment.Ended`
  - `FoTask.Created` / `FoTask.Updated`

#### 4.2 Trip Monitor
- Current dependency:
  - `IoT.Heartbeat` (uses `operationState`)
- Update to consume:
  - `Device.Heartbeat`
  - `Trip.Started` / `Trip.Completed`

#### 4.3 Operations Service
- Current dependencies:
  - `Robots.StateChange` (for `RobotStateHistory`)
- Update to consume:
  - `Trip.*` events
  - `Deployment.*` events
  - `FoTask.*` events
  - `Device.*` events

#### 4.4 Deliveries Service
- Current dependencies:
  - `Robots.LidOpen`, `Robots.LidClose`
- Update to consume:
  - `Device.LidOpened`, `Device.LidClosed`
  - `Device.CargoChanged` (if available)

---

### Phase 5: Event Compatibility Strategy

**Goal:** Ensure the system continues to work while transitioning.

1. Dual-publish legacy + new events for a full release cycle.
2. Update consumers in priority order:
   1. Dispatch Engine
   2. Trip Monitor
   3. Operations Service
   4. Deliveries Service
3. Add log-level comparison where both old and new events are available.
4. Only after all consumers migrate, begin deprecating legacy events.

---

### Phase 6: Work Breakdown Checklist

**1. Event Definitions**
- Add Device event interfaces in `service/device`.
- Register names and payload types in common exchange definitions.

**2. Publishers**
- Update State Service to dual-publish Device events.
- Update Operations/Deliveries producers to publish domain-specific events.

**3. Consumers**
- Migrate Dispatch Engine to Device + domain events.
- Migrate Trip Monitor to Device + Trip events.
- Migrate Operations and Deliveries to Device + domain events.

**4. Validation**
- Confirm all existing event payload fields are available via new events.
- Confirm no consumer requires `operationState` for POC.

---

### Open Questions for Hand-off
1. Which service should own the new `Trip.*` event exchange definitions?
2. Do we need separate exchanges per domain (Device, Trip, Deployment), or reuse the existing `Robots` exchange?
3. Should we keep `Robots.StateChange` as an audit event after migration, or remove it entirely?
