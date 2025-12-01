# Log-Driven Analytical Metrics - Implementation Plan

## 🤖 Instructions for AI Agent

**Role**: You are the Implementer. Your goal is to execute this plan one task at a time, ensuring high quality and strict adherence to the design.

**Protocol**:
1.  **State Check**: Read this file. Find the first unchecked task (`- [ ]`). This is your current objective. Do *not* work on multiple tasks at once.
2.  **Context Gathering**: Read the "Context & References" links if you are a new agent starting a session. Understand the "Golden Thread" of IDs.
3.  **Execution**:
    - Follow the "Exact Changes" precisely.
    - If you encounter code that differs significantly from expectation, stop and ask the user for guidance.
4.  **Verification (Mandatory)**:
    - Run the specified verification commands (tests, scripts, or manual checks).
    - **Self-Correction**: If verification fails, analyze the error, fix the code, and re-run verification. Do not ask for review until verification passes.
5.  **Review Request**:
    - Once verification passes, present the results (command output, logs) to the user.
    - Ask: *"Task X.X is verified. Shall I mark it as done and proceed?"*
6.  **Completion**:
    - Only after User Approval: Update this file to mark the task as `[x]`.
    - (Optional) Ask the user if they want to commit the changes to Git before moving on.

## 📂 Context & References
- **Business Proposal**: [Log-Driven OLAP Metrics - Business Proposal.md](Log-Driven%20OLAP%20Metrics%20-%20Business%20Proposal.md)
- **Engineering Spec**: [Log-Driven OLAP Metrics - Engineering Spec.md](Log-Driven%20OLAP%20Metrics%20-%20Engineering%20Spec.md)
- **Deep Research**: [Observability Lifecycle Analysis](../research/Observability_Lifecycle_Analysis.md)
- **Registry Path**: `delivery-platform/lib/common/src/events/registry.ts` (You will create this)
- **Infrastructure Path**: `coco-infra/environments/{env}/core/{region}/eks/`

---

## Milestone 1: Foundation & Registry
*Goal: Establish the contract for analytics events. Each task is a small, reviewable PR.*

### Task 1.1: Registry Skeleton & Context Interface
- **Goal**: Create the `AnalyticsContext` interface and basic file structure.
- **Files**: `delivery-platform/lib/common/src/events/registry.ts`
- **Exact Changes**:
    1.  Create the file `delivery-platform/lib/common/src/events/registry.ts`.
    2.  Export a `Events` constant object (empty for now).
    3.  Export a `EventType` type derived from the values of `Events`.
    4.  Export an interface `AnalyticsContext` with the following optional properties:
        -   `event`: `EventType` (required)
        -   `timestamp`: `string` (required)
        -   `merchant_id`: `string`
        -   `delivery_id`: `string`
        -   `demand_id`: `string`
        -   `trip_id`: `string`
        -   `robot_id`: `string`
        -   `pilot_id`: `string`
        -   `[key: string]: any` (index signature for flexibility)
- **Verification**:
    1.  Run `tsc --noEmit` in the terminal to ensure no syntax errors.
    2.  Create a temporary file `verify_registry.ts` with content:
        ```typescript
        import { AnalyticsContext, Events } from './delivery-platform/lib/common/src/events/registry';
        const ctx: AnalyticsContext = { event: 'test' as any, timestamp: new Date().toISOString() };
        console.log('Registry loads:', !!Events);
        ```
    3.  Run `ts-node verify_registry.ts` and expect output: `Registry loads: true`.
    4.  Delete the temporary file.
- **Status**: [ ]

### Task 1.2: Extend Logger Interface
- **Goal**: Add `event()` method to the shared Logger to enforce structured event logging.
- **Files**:
    - `delivery-platform/lib/common/src/logging/interfaces.ts`
    - `delivery-platform/lib/common/src/logging/adapters/json-logger.ts`
    - `delivery-platform/lib/common/src/logging/adapters/default-logger.ts`
- **Exact Changes**:
    1.  **Update `ILogger` in `interfaces.ts`**:
        -   Add abstract method: `abstract event(name: string, context: Record<string, any>): void;`
    2.  **Update `JsonLogger` in `json-logger.ts`**:
        -   Implement `event(name, context)`:
            ```typescript
            event(name: string, context: Record<string, any>) {
              this.log({ event: name, ...context });
            }
            ```
    3.  **Update `DefaultLogger` in `default-logger.ts`**:
        -   Implement `event(name, context)`:
            ```typescript
            event(name: string, context: Record<string, any>) {
              this.log(`[Event: ${name}]`, context);
            }
            ```
- **Verification**:
    1.  Run `tsc --noEmit` to check for interface implementation errors.
    2.  Create `verify_logger.ts`:
        ```typescript
        import { JsonLogger } from './delivery-platform/lib/common/src/logging/adapters/json-logger';
        const logger = new JsonLogger();
        const spy = jest.spyOn(logger, 'log');
        logger.event('test_event', { foo: 'bar' });
        console.log('Called with:', spy.mock.calls[0][0]);
        ```
    3.  Run `ts-node verify_logger.ts` (ensure jest/ts-node setup allows spying, or just use `console.log` override). Expect output containing `event: 'test_event'`.
- **Status**: [ ]

### Task 1.3: Robot Lifecycle Events
- **Goal**: Add robot-specific events to the registry.
- **Files**: `delivery-platform/lib/common/src/events/registry.ts`
- **Exact Changes**:
    1.  Update the `Events` object to include the following keys and values:
        ```typescript
        export const Events = {
          ROBOT_STATE_CHANGED: 'robot.state_changed',
          ROBOT_DEPLOYED: 'robot.deployed',
          ROBOT_PARKED: 'robot.parked',
          ROBOT_GROUNDED: 'robot.grounded',
          ROBOT_OFF_DUTY: 'robot.off_duty',
        } as const;
        ```
    2.  Add JSDoc comments to each event explaining usage (e.g., "Fired when robot operational state changes").
- **Verification**:
    1.  Run `tsc --noEmit`.
    2.  Create `verify_events.ts` logging `Events.ROBOT_STATE_CHANGED`.
    3.  Run `ts-node verify_events.ts`. Output must be `robot.state_changed`.
    4.  Delete the file.
- **Status**: [ ]

### Task 1.3: Demand & Delivery Events
- **Goal**: Add demand and delivery events.
- **Files**: `delivery-platform/lib/common/src/events/registry.ts`
- **Exact Changes**:
    1.  Add these entries to `Events`:
        ```typescript
        // Demand
        DEMAND_CREATED: 'demand.created',
        DEMAND_ASSIGNED: 'demand.assigned',
        DEMAND_ACTIVE: 'demand.active',
        DEMAND_FULFILLED: 'demand.fulfilled',
        DEMAND_CANCELED: 'demand.canceled',
        DEMAND_REPLAN_ORPHANED: 'demand.replan.orphaned',
        DEMAND_REPLAN_REHOMED: 'demand.replan.rehomed',
        
        // Delivery
        DELIVERY_PICKUP_ARRIVAL: 'delivery.pickup_arrival',
        DELIVERY_PICKUP_COMPLETED: 'delivery.pickup_completed',
        DELIVERY_DROPOFF_COMPLETED: 'delivery.dropoff_completed',
        DEMAND_CONFIRMED: 'demand.confirmed', // Equivalent to Quote Accepted
        ```
- **Verification**:
    1.  Run `tsc --noEmit`.
    2.  Check file content to ensure keys match exactly.
- **Status**: [ ]

### Task 1.4: Trip & Pilot Events
- **Goal**: Add trip and pilot events.
- **Files**: `delivery-platform/lib/common/src/events/registry.ts`
- **Exact Changes**:
    1.  Add these entries to `Events`:
        ```typescript
        // Trip
        TRIP_CREATED: 'trip.created',
        TRIP_STARTED: 'trip.started',
        TRIP_COMPLETED: 'trip.completed',
        TRIP_ACTIVATED: 'trip.activated',
        
        // Pilot
        PILOT_SHIFT_STARTED: 'pilot.shift_started',
        PILOT_SHIFT_ENDED: 'pilot.shift_ended',
        PILOT_ASSIGNMENT_STARTED: 'pilot.assignment_started', // Assignment logic start
        PILOT_ASSIGNED: 'pilot.assigned', // Successful match
        ```
- **Verification**:
    1.  Run `tsc --noEmit`.
- **Status**: [ ]

---

## Milestone 2: Context Propagation (The "Golden Thread")
*Goal: Update core services to log events with the required IDs. Each service update is a separate PR.*

### Task 2.1: Instrument Dispatch Engine (Demand)
- **Goal**: Log `DEMAND_CREATED` and `DEMAND_ASSIGNED` with `merchant_id`.
- **Files**:
    - `delivery-platform/service/dispatch-engine/src/modules/demand/service/demand.service.ts`
    - `delivery-platform/service/dispatch-engine/src/modules/demand/repo/demand.repo.ts`
- **Exact Changes**:
    1.  **Modify `createDemand` in `demand.service.ts`**:
        -   Import `Events` from registry.
        -   After `this.repo.create(...)` returns successfully:
        -   Extract `merchantId` from `params.features` (cast as necessary).
        -   Call `this.logger.info` with object:
            ```typescript
            {
              event: Events.DEMAND_CREATED,
              demand_id: result.id,
              merchant_id: merchantId, // Ensure this is extracted safely
              timestamp: new Date().toISOString()
            }
            ```
    2.  **Modify `assignRobot` in `demand.repo.ts`**:
        -   Import `Events` from registry.
        -   After successful assignment (inside the transaction or immediately after):
        -   Call `this.logger.info`:
            ```typescript
            {
              event: Events.DEMAND_ASSIGNED,
              demand_id: demandId,
              robot_id: serial,
              timestamp: new Date().toISOString()
            }
            ```
- **Verification**:
    1.  Run `turbo run test --filter=dispatch-engine`.
    2.  **Crucial**: Inspect the test console output. You MUST see the JSON log line containing `"event": "demand.created"` and `"merchant_id"`. If existing tests don't trigger it, modify a test in `demand.service.spec.ts` to verify the spy on `logger.info` was called with these exact parameters.
- **Status**: [ ]

### Task 2.2: Instrument Deliveries Service (Quote/Delivery)
- **Goal**: Log `QUOTE_ACCEPTED` (implied by `DEMAND_CONFIRMED`) and Delivery transitions.
- **Files**:
    - `delivery-platform/service/deliveries/src/modules/providers/robot/robot.service.ts`
    - `delivery-platform/service/deliveries/src/modules/delivery/service/delivery.service.ts`
- **Exact Changes**:
    1.  **Modify `request` method in `robot.service.ts`**:
        -   Find where `this.engine.confirmDemand` is called.
        -   Log `Events.DEMAND_CONFIRMED` with `quote_id: quote.id` and `merchant_id: params.merchantId`.
    2.  **Modify `DeliveryService` methods**:
        -   In methods handling pickup/dropoff completion (search for status updates to `PickedUp` or `Delivered`), log `Events.DELIVERY_PICKUP_COMPLETED` and `Events.DELIVERY_DROPOFF_COMPLETED`.
        -   Ensure `delivery_id` is included in the log payload.
- **Verification**:
    1.  Run `turbo run test --filter=deliveries`.
    2.  Grep output for `demand.confirmed` and `delivery.dropoff_completed`.
- **Status**: [ ]

### Task 2.3: Instrument Operations Service (Trip/Pilot)
- **Goal**: Log Trip and Pilot Assignment events with full context.
- **Files**:
    - `delivery-platform/service/operations/src/modules/trips/services/pilot-trips.service.ts`
    - `delivery-platform/service/operations/src/modules/pilots/pilot-assignments/repositories/repo.service.ts`
- **Exact Changes**:
    1.  **Modify `activatePilotTrip` in `pilot-trips.service.ts`**:
        -   Log `Events.TRIP_ACTIVATED` with `trip_id: tripId` and `robot_id: botSerial`.
    2.  **Modify `savePilotAssignedToTrip` in `repo.service.ts`**:
        -   This method already has `meta` containing `deliveryId`.
        -   Log `Events.PILOT_ASSIGNED` with:
            -   `trip_id`: `tripId`
            -   `pilot_id`: `pilotId`
            -   `delivery_id`: `meta.deliveryId` (Critical for joining!)
- **Verification**:
    1.  Run `turbo run test --filter=operations`.
    2.  Verify `trip.activated` and `pilot.assigned` appear in logs with the correct IDs.
- **Status**: [ ]

### Task 2.4: Instrument Replan Service (Orphan/Rehome)
- **Goal**: Capture robot switch events during replanning.
- **Files**: `delivery-platform/service/dispatch-engine/src/modules/planner/service/replan.service.ts`
- **Exact Changes**:
    1.  **Modify `rehomeOrphanedDelivery`**:
        -   After successful re-assignment, log `Events.DEMAND_REPLAN_REHOMED`.
        -   Include: `demand_id: request.requestId`, `robot_id: estimate.deviceId` (the *new* robot).
    2.  **Modify `orphanDelivery`**:
        -   Log `Events.DEMAND_REPLAN_ORPHANED`.
        -   Include: `demand_id: request.requestId`, `robot_id: request.deviceId` (the *old* robot).
- **Verification**:
    1.  Run `turbo run test --filter=dispatch-engine -- --testPathPattern=replan.service.spec.ts`.
    2.  These flows are complex; ensure the test actually triggers the "orphan" logic. You may need to force a mock failure in the test setup to see the log.
- **Status**: [ ]

### Task 2.5: Robot State Gap
- **Goal**: Ensure Robot State changes can be linked to trips.
- **Files**: `delivery-platform/service/operations/src/modules/robots/handlers/robot-state-change-handler.ts`
- **Exact Changes**:
    1.  Examine `StateChangeEvent` payload.
    2.  If `tripId` is missing, log a warning or a new event `Events.ROBOT_STATE_CONTEXT` that includes `robot_id` and `trip_id` if available in the handler's scope (e.g., by fetching the robot's current state from DB).
    3.  *Decision*: If fetching DB state is too expensive on every event, document this limitation in `obsidian/research/Observability_Lifecycle_Analysis.md` and rely on time-window joins in dbt.
- **Verification**:
    -   Document the decision in the Research doc.
- **Status**: [ ]

---

## Milestone 3: Infrastructure & Ingestion
*Goal: Deploy log shipping pipeline. Separated by resource type.*

### Task 3.1: S3 Bucket & Permissions
- **Goal**: Create the S3 bucket for cold storage.
- **Files**: `coco-infra/modules/core/s3/` (or equivalent Terraform module path)
- **Exact Changes**:
    1.  Add a Terraform resource `aws_s3_bucket` named `analytics_logs`.
    2.  Set bucket name to `coco-event-logs-{env}`.
    3.  Add `lifecycle_rule`:
        -   `id`: "archive_logs"
        -   `enabled`: true
        -   `transition`: days = 90, storage_class = "STANDARD_IA"
        -   `expiration`: days = 730 (2 years)
    4.  Create `aws_iam_policy` `vector_s3_write` allowing `s3:PutObject` on this bucket.
- **Verification**:
    1.  Run `terraform plan`.
    2.  **Human Required**: Verify output shows `+ aws_s3_bucket` creation. Ensure it does *not* modify existing critical buckets.
- **Status**: [ ]

### Task 3.2: Vector Configuration (Filtering)
- **Goal**: Configure Vector to filter logs and ship "business events" to S3.
- **Files**: `coco-infra/environments/{env}/core/{region}/eks/vector-values.yaml`
- **Exact Changes**:
    1.  Locate the `transforms` section. Add `filter_events`:
        ```yaml
        type: filter
        inputs: [kubernetes_logs]
        condition: exists(.event)
        ```
    2.  Locate the `sinks` section. Add `s3_archive`:
        ```yaml
        type: aws_s3
        inputs: [filter_events]
        bucket: "coco-event-logs-{{ .Values.env }}"
        region: "{{ .Values.region }}"
        key_prefix: "dt=%Y-%m-%d/"
        encoding:
          codec: json
        compression: gzip
        ```
- **Verification**:
    1.  Run `terraform plan` for the EKS module.
    2.  Verify `values.yaml` changes are present.
    3.  **Human Required**: Apply changes to `dev` cluster. Manually emit a log with `event: "test"`. Check S3 bucket after ~5 minutes (buffer time) for a `.gz` file containing that log.
- **Status**: [ ]

### Task 3.3: Redshift External Schema
- **Goal**: Allow Redshift to query S3 data.
- **Files**: `analytics/migrations/V1__create_external_schema.sql` (New file)
- **Exact Changes**:
    1.  Create the SQL file.
    2.  Add command to create external schema:
        ```sql
        create external schema if not exists spectrum
        from data catalog
        database 'spectrum_db'
        iam_role 'arn:aws:iam::...:role/RedshiftRole'
        create external database if not exists;
        ```
    3.  Add command to create external table `raw_events`:
        ```sql
        create external table spectrum.raw_events (
          event_data super
        )
        stored as json
        location 's3://coco-event-logs-prod/';
        ```
- **Verification**:
    1.  **Human Required**: Run the SQL in a Redshift client.
    2.  Run query: `SELECT count(*) FROM spectrum.raw_events;`. It should succeed (even if count is 0).
- **Status**: [ ]

---

## Milestone 4: Analytical Models (dbt)
*Goal: Turn raw logs into metrics.*

### Task 4.1: dbt Source & Staging
- **Goal**: Define the source table in dbt.
- **Files**:
    - `analytics/dbt/models/sources.yml`
    - `analytics/dbt/models/staging/stg_events.sql`
- **Exact Changes**:
    1.  In `sources.yml`, define source `redshift` with table `raw_events`.
    2.  In `stg_events.sql`, select from source:
        ```sql
        SELECT
            event_data.event::varchar as event_name,
            event_data.timestamp::timestamp as occurred_at,
            event_data.merchant_id::varchar as merchant_id,
            -- ... extract other fields
        FROM {{ source('redshift', 'raw_events') }}
        ```
- **Verification**:
    1.  Run `dbt compile`.
    2.  Run `dbt run --select stg_events`.
- **Status**: [ ]

### Task 4.2: Delivery Lifecycle Fact
- **Goal**: Join Demand, Delivery, and Trip events.
- **Files**: `analytics/dbt/models/marts/core/fact_delivery_lifecycle.sql`
- **Exact Changes**:
    1.  Create CTEs for `demand_events` (filter `demand.*`), `delivery_events` (`delivery.*`), `trip_events`.
    2.  Join them on `delivery_id`.
    3.  Calculate durations: `pickup_duration_seconds = datediff(second, pickup_arrival, pickup_completed)`.
- **Verification**:
    1.  Run `dbt compile`.
    2.  Run `dbt run --select fact_delivery_lifecycle`.
    3.  **Human Required**: Inspect rows in the output table. Confirm `delivery_id`s match known deliveries.
- **Status**: [ ]

### Task 4.3: Robot Utilization Model
- **Goal**: Calculate time-in-state.
- **Files**: `analytics/dbt/models/marts/operations/dim_robot_state.sql`
- **Exact Changes**:
    1.  Select `robot_id`, `timestamp`, `state` from `stg_events` where event is `robot.state_changed`.
    2.  Use `LEAD(timestamp) OVER (PARTITION BY robot_id ORDER BY timestamp)` to get `next_state_start`.
    3.  Calculate `duration = next_state_start - timestamp`.
- **Verification**:
    1.  Run `dbt compile`.
    2.  Run `dbt run --select dim_robot_state`.
    3.  **Human Required**: Sanity check: `SELECT SUM(duration) FROM dim_robot_state WHERE robot_id='X'` should roughly equal the elapsed time.
- **Status**: [ ]
