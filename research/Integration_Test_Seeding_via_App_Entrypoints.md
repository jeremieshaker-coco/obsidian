# Research: Integration Test Seeding via App Entrypoints

## Summary

This document analyzes the current testing patterns in `dispatch-engine` and identifies opportunities to replace direct Prisma seeding with application entrypoint-based record creation, leveraging patterns already established in the simulation (`sim`) module.

---

## Current State: demand.integration.spec.ts

### Pattern Overview

The demand integration tests currently seed supply data (robots and pilots) **directly into Prisma**, bypassing the application layer entirely:

```typescript
// test/utils/seed.ts
export async function seedRobots(prisma, robotIds, coords, now, parkingLotId, overrides) {
  for (const robotId of robotIds) {
    await prisma.robot.upsert({
      where: { id: robotId },
      update: {},
      create: createTestRobot(now, { id: robotId, lat, lng, deployedLocationId, ...overrides }),
    });
  }
}

export async function seedPilots(prisma, pilotIds, now, overrides) {
  for (const pilotId of pilotIds) {
    await prisma.resource.upsert({
      where: { identifier: pilotId },
      update: {},
      create: { identifier: pilot.id, type: 'Pilot', data: pilot },
    });
  }
}
```

### Problems with Direct Prisma Seeding

1. **Bypasses Application Logic**: The `SupplyService.processRobotSupply()` and `processPilotSupply()` methods contain business logic that transforms events into DB records. Direct seeding skips this.

2. **Schema Drift Risk**: If the supply processing logic changes (e.g., new required fields, default behaviors), tests using direct seeding won't catch regressions.

3. **Inconsistent Test Data**: The data shape created by factories may not match what the application actually produces, leading to false positives.

4. **Maintenance Burden**: Two separate code paths to maintain—one for production and one for tests.

---

## Sim Module Architecture

The simulation module provides a mature pattern for seeding data through application entrypoints.

### Key Components

#### 1. MessageService — Internal Event Router

The `MessageService` acts as an in-memory AMQP replacement, routing events to registered handlers:

```typescript
// src/sim/modules/message/message.service.ts
@Injectable()
export class MessageService {
  consumers: Consumer[];

  async publish<E>(exchange: E, payload: Exchanges.PayloadOf<E>) {
    const consumers = this.consumers.filter((c) => c.exchange === ex);
    for (const c of consumers) {
      await c.handlerRef.call(c.instanceRef, payload);  // Direct handler invocation
    }
  }
}
```

#### 2. Supply Models — Emit Events Instead of Direct DB Writes

The `PilotModel` and `RobotModel` emit supply events via callbacks:

```typescript
// src/sim/models/pilot.model.ts
export class PilotModel extends Model<State, FsmState> {
  constructor(
    id: string,
    now: Date,
    schedule: Duration[],
    private readonly emit: (d: DispatchSupplyEvent) => Promise<void>,  // Event emission callback
  ) { ... }

  private async syncPilot({ fsmState, force }: SyncParams): Promise<boolean> {
    const supplyEvent: PilotSupplyEventPayload = {
      pilotId: this.id,
      pilotCallCenter: PilotCallCenter.Colombia,
      type: ResourceType.Pilot,
      timestamp: this.now.getTime(),
      shiftStatus: this.fsmStateToShiftStatus[fsmState],
      experienceLevel: PilotExperienceLevel.INTERMEDIATE,
      schedule: this.schedule,
      ...
    };
    await this.emit({ supply: supplyEvent, source, scope, type });  // Emits through app entrypoint
    return true;
  }
}
```

#### 3. SupplyEventHandler — The Application Entrypoint

Events are processed by the `SupplyEventHandler`, which calls `SupplyService`:

```typescript
// src/modules/supply/handlers/supply-event.handler.ts
@Injectable()
export class SupplyEventHandler {
  @Subscribe(Exchanges.Names.DispatchEngine.SupplyEvent, '#', 'dispatch-engine/handle-supply-event')
  async handleSupplyEvent(event: DispatchSupplyEvent): Promise<void> {
    const res = 'robotSerial' in event.supply
      ? await this.supply.processRobotSupply(event.supply)
      : await this.supply.processPilotSupply(event.supply);
    // ... error handling
  }
}
```

#### 4. Demand Creation — Uses DemandController Directly

The sim creates demands through the actual API layer:

```typescript
// src/sim/util.ts
export async function genDeliveryOpportunity(params: {
  api: DemandController;
  pickupTime: Date;
  merchantId: string;
  destination: { latitude: number; longitude: number };
  attemptId: string;
  deliveryId: string;
}): Promise<Demand | Error | RejectReason> {
  const request = quoteRequest({ ... });
  const estimate = await api.estimate(request);  // Uses real controller
  const quote = await api.quote(request);        // Uses real controller
  const demand = await api.confirm(quote.id);    // Uses real controller
  return demand;
}
```

#### 5. Prismock — In-Memory Database

The sim uses `prismock` for an in-memory Prisma-compatible database:

```typescript
// src/prisma/prisma.module.ts
@Module({
  providers: [{
    provide: PrismaService,
    useClass: SimEnabled ? PrismockService : PrismaService,
  }],
})
export class PrismaModule {}
```

---

## Event Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              SIMULATION / TEST                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐     ┌──────────────────┐     ┌───────────────────────┐  │
│  │   PilotModel    │ ──▶ │  MessageService  │ ──▶ │  SupplyEventHandler   │  │
│  │   RobotModel    │     │    (pub/sub)     │     │                       │  │
│  └─────────────────┘     └──────────────────┘     └───────────────────────┘  │
│         │                                                    │               │
│         │ emit()                                             │               │
│         ▼                                                    ▼               │
│  ┌─────────────────┐                              ┌───────────────────────┐  │
│  │ DispatchSupply  │                              │    SupplyService      │  │
│  │     Event       │                              │  processRobotSupply() │  │
│  └─────────────────┘                              │  processPilotSupply() │  │
│                                                   └───────────────────────┘  │
│                                                              │               │
│                                                              ▼               │
│                                                   ┌───────────────────────┐  │
│                                                   │     SupplyRepo        │  │
│                                                   │   (Prisma/Prismock)   │  │
│                                                   └───────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Reuse Opportunities

### 1. Create Test Helpers That Emit Supply Events

Instead of direct Prisma writes, create helpers that emit supply events:

```typescript
// Proposed: test/utils/supply-helpers.ts
export async function seedPilotViaEvent(
  supplyHandler: SupplyEventHandler,
  params: { pilotId: string; schedule: Duration[]; status: PilotShiftStatus }
): Promise<Pilot> {
  const event: DispatchSupplyEvent = {
    supply: {
      pilotId: params.pilotId,
      pilotCallCenter: PilotCallCenter.Colombia,
      type: ResourceType.Pilot,
      timestamp: Date.now(),
      shiftStatus: params.status,
      experienceLevel: PilotExperienceLevel.INTERMEDIATE,
      schedule: params.schedule,
    },
    source: EventSinkSource.OPERATIONS_PILOTS,
    scope: EventScope.EXTERNAL,
    type: EventType.Supply,
  };
  await supplyHandler.handleSupplyEvent(event);
  // Return the created pilot via SupplyService.pilots() or repo query
}

export async function seedRobotViaEvent(
  supplyHandler: SupplyEventHandler,
  params: { serial: string; location: Location; deployed: ParkingLotLocation; healthy: boolean }
): Promise<Robot> {
  const event: DispatchSupplyEvent = {
    supply: {
      robotSerial: params.serial,
      location: params.location,
      battery: 100,
      healthy: params.healthy,
      type: ResourceType.Robot,
      timestamp: Date.now(),
      // ... other fields
    },
    source: EventSinkSource.OPERATIONS_ROBOTS,
    scope: EventScope.EXTERNAL,
    type: EventType.Supply,
  };
  await supplyHandler.handleSupplyEvent(event);
}
```

### 2. Reuse Sim Models for Stateful Test Fixtures

The `PilotModel` and `RobotModel` classes manage state transitions (on-shift, on-break, off-shift). These could be reused to create realistic test fixtures:

```typescript
// Example: Creating a pilot that goes through shift transitions
const pilot = new PilotModel('test-pilot', now, schedule, async (evt) => {
  await supplyHandler.handleSupplyEvent(evt);
});
await pilot.tick(now);  // Syncs to DB via event handler
```

### 3. Use MessageService as Test Double for AMQP

The `MessageService` can replace `FakeAmqpPublisher` in integration tests:

```typescript
// Instead of:
.overrideProvider(AmqpPublisher).useValue(fakeAmqpPublisher)

// Use:
.overrideProvider(AmqpPublisher).useClass(MessageService)
```

This would route published messages directly to their handlers within the same process.

### 4. Shared Factory Functions

Extract the event payload construction from sim models into shared factories:

```typescript
// Proposed: shared/test-utils/supply-event-factories.ts
export function createPilotSupplyEvent(params: Partial<PilotSupplyEventPayload>): DispatchSupplyEvent {
  return {
    supply: {
      pilotId: params.pilotId ?? uuid(),
      pilotCallCenter: params.pilotCallCenter ?? PilotCallCenter.Colombia,
      type: ResourceType.Pilot,
      timestamp: params.timestamp ?? Date.now(),
      shiftStatus: params.shiftStatus ?? PilotShiftStatus.ON_SHIFT,
      experienceLevel: params.experienceLevel ?? PilotExperienceLevel.INTERMEDIATE,
      schedule: params.schedule ?? [{ start: new Date(), end: addHours(new Date(), 8) }],
    },
    source: EventSinkSource.OPERATIONS_PILOTS,
    scope: EventScope.EXTERNAL,
    type: EventType.Supply,
  };
}
```

---

## Migration Path

### Phase 1: Create Event-Based Helpers

1. Create `test/utils/supply-via-events.ts` with helper functions
2. Add these as alternative seeding methods alongside existing Prisma-direct helpers
3. Validate parity between both approaches

### Phase 2: Migrate Integration Tests

1. Update `demand.integration.spec.ts` to use event-based seeding
2. Remove direct Prisma imports from test setup functions
3. Verify all tests still pass

### Phase 3: Deprecate Direct Seeding

1. Remove `seedRobots` and `seedPilots` from `test/utils/seed.ts`
2. Update other integration tests to use new pattern
3. Document the new pattern in the test README

---

## Benefits

| Aspect | Direct Prisma Seeding | Event-Based Seeding |
|--------|----------------------|---------------------|
| Tests Application Logic | ❌ No | ✅ Yes |
| Schema Drift Detection | ❌ No | ✅ Yes |
| Maintenance | High (dual code paths) | Low (single path) |
| Test Realism | Low | High |
| Code Reuse | Limited | Leverages sim module |

---

## Open Questions

1. **Performance**: Will event-based seeding be slower than direct Prisma? (Likely negligible for integration tests)

2. **Prismock vs Real DB**: Should integration tests also use prismock, or continue with real Postgres in Docker?

3. **Async Handling**: Some supply processing is async (e.g., hydration from operations). How to handle in tests?

4. **Partial Adoption**: Can we migrate incrementally, or does the pattern require all-or-nothing?

---

## References

- `src/sim/sim.service.ts` — Simulation orchestrator
- `src/sim/models/pilot.model.ts` — Pilot model with event emission
- `src/sim/models/robot.model.ts` — Robot model with event emission
- `src/sim/modules/message/message.service.ts` — Internal event routing
- `src/modules/supply/handlers/supply-event.handler.ts` — Event handler entrypoint
- `src/modules/supply/service/supply.service.ts` — Supply processing logic
- `test/demand/demand.integration.spec.ts` — Current integration test pattern











