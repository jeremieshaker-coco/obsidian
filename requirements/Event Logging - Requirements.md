# Event Logging: Product Requirements

## Objective

Unify our observability and analytics systems by making structured logs the single source of truth for both real-time monitoring (Datadog) and business analytics (Redshift).

## Current State

**What We Have**:
- ✅ Structured JSON logging
- ✅ Datadog Agent collecting all container logs
- ✅ StatsD metrics for real-time counters/timers
- ✅ Redshift data warehouse with Analytical Metrics

**The Gap**:
- ❌ Analytical metrics are derived from analytics events, not logs
- ❌ No automated pipeline from logs → Redshift
- ❌ When metrics show anomalies, we can't trace back to the event stream
- ❌ Root cause analysis is slow (manually reading Datadog logs to reconstruct event sequences)
- ❌ No path for metric discovery (can't query historical logs to find patterns)

---

## Functional Requirements

### FR-1: Event Logging API

The system shall provide a dedicated logging method (`logger.logEvent()`) for emitting analytics events.

**FR-1.1**: Events shall be logged at `INFO` level by default.

**FR-1.2**: If the `error` field is populated in the payload, the event shall automatically be logged at `ERROR` level.

**FR-1.3**: All events shall include the `event` property in the JSON output, enabling filtering by the ingestion pipeline.

### FR-2: Event Registry

The system shall maintain a centralized Event Registry as the single source of truth for all event identifiers.

**FR-2.1**: All analytics events MUST use identifiers from the registry (no free-form strings).

**FR-2.2**: Event identifiers shall follow the naming convention: `category.action` (dot notation).

**FR-2.3**: The registry shall be importable in TypeScript and documented for Go usage.

### FR-3: Type-Safe Event Payloads

The system shall enforce type-safe event payloads at compile time (TypeScript).

**FR-3.1**: Each event type MAY have a specific payload schema with required fields.

**FR-3.2**: If no specific schema is defined for an event, a default schema with optional fields shall be used.

**FR-3.3**: Business-critical events (e.g., `offer.rejected`) SHALL have required fields enforced (e.g., `rejection_reason`).

### FR-4: Event Categories

The system shall support the following event categories:

#### FR-4.1: Demand Inbound (Offer Funnel)
Events tracking demand BEFORE we accept it:
- `offer.received` - Partner sends a delivery request
- `offer.accepted` - We accepted the offer
- `offer.rejected` - We rejected (must include `rejection_reason`)
- `offer.expired` - We didn't respond in time
- `quote.requested` - Partner asks for ETA/price estimate
- `quote.provided` - We respond with quote

#### FR-4.2: Robot Supply Lifecycle
- `robot.state_changed`
- `robot.deployed`
- `robot.parked`
- `robot.grounded`
- `robot.off_duty`

#### FR-4.3: Demand (Order) Lifecycle
Events tracking demand AFTER we've accepted it:
- `demand.created`
- `demand.assigned`
- `demand.active`
- `demand.fulfilled`
- `demand.canceled`
- `demand.etas_updated`

#### FR-4.4: Trip Execution Lifecycle
- `trip.created`
- `trip.assigned`
- `trip.started`
- `trip.completed`
- `trip.canceled`

#### FR-4.5: Pilot Lifecycle
- `pilot.shift_started`
- `pilot.shift_ended`
- `pilot.assignment_started`
- `pilot.assignment_ended`

#### FR-4.6: Delivery Execution Lifecycle
- `delivery.pickup_arrival`
- `delivery.pickup_started`
- `delivery.pickup_completed`
- `delivery.dropoff_arrival`
- `delivery.dropoff_started`
- `delivery.dropoff_completed`
- `delivery.failed`

#### FR-4.7: Attempt Lifecycle
- `attempt.started`
- `attempt.completed`
- `attempt.failed`
- `attempt.rescued` - Handed off to human courier

### FR-5: Event Payload Standards

**FR-5.1**: Every event MUST include:
- `event` - Event identifier from the registry
- `timestamp` - ISO 8601 format

**FR-5.2**: Every event SHOULD include (when available):
- `source` - Partner origin (`uber`, `doordash`, `wolt`, `direct`)
- Entity IDs: `robot_id`, `delivery_id`, `demand_id`, `trip_id`, `pilot_id`, `merchant_id`, `offer_id`, `attempt_id`

**FR-5.3**: Location context SHOULD be included for demand analysis:
- `parking_lot_id` - Where robot is stationed
- `zone_id` - Geographic zone for aggregation

**FR-5.4**: Offer rejection events MUST include:
- `rejection_reason` - Why we said no (e.g., `no_available_robots`, `route_too_long`)

**FR-5.5**: Error events MUST include:
- `error.code` - Machine-readable error code
- `error.message` - Human-readable description

### FR-6: Event Ingestion Pipeline

**FR-6.1**: Events shall be filtered at the source (only logs with `event` property ingested to analytics).

**FR-6.2**: Events shall be stored in a queryable data warehouse (Redshift).

**FR-6.3**: Events shall be partitioned by date for efficient time-range queries.

**FR-6.4**: Ingestion shall occur at least hourly.

### FR-7: Sessionization

**FR-7.1**: The system shall support reconstructing the complete event stream for a delivery ("delivery session").

**FR-7.2**: Events without `delivery_id` (e.g., robot hardware errors) shall be linkable via `robot_id` and time windows.

**FR-7.3**: The system shall handle re-assignments (Robot A → Robot B) by respecting time windows.

---

## Non-Functional Requirements

### NFR-1: Performance
- Event logging shall not block application code (asynchronous)
- Ingestion pipeline latency: < 1 hour from log emission to queryable in Redshift

### NFR-2: Cost
- Target: < $250/month for infrastructure
- No unbounded cost growth from debug logs (smart filtering)

### NFR-3: Reliability
- Pipeline failures shall not affect production logging (Datadog continues working)
- Idempotent ingestion (can re-run for past dates)

### NFR-4: Developer Experience
- TypeScript autocomplete for event names
- Compile-time enforcement of required fields for critical events
- Clear documentation of all events in a catalog

---

## Analytics Capabilities (Success Criteria)

### Supply-Side Analytics
- **SLA Tracking**: Time from demand created → robot moving
- **Wait Time**: Time robots spend waiting at merchants
- **Root Cause**: Identify exactly where failures occur in the delivery lifecycle

### Demand-Side Analytics
- **Acceptance Rate**: `offer.accepted / offer.received` by source, location, time
- **Rejection Analysis**: Group by `rejection_reason` to understand missed demand
- **Location Patterns**: Demand volume and acceptance rates by `parking_lot_id`
- **Partner Comparison**: Uber vs. DoorDash acceptance rates

### Operational
- Delivery session reconstruction in < 30 seconds (vs. 20-30 minutes manual)
- Ad-hoc metric discovery from historical data

---

## Out of Scope

- Changes to StatsD metrics (real-time counters/timers)
- Changes to RabbitMQ event bus
- Full log ingestion (only `event`-tagged logs)
- Real-time streaming analytics (batch/hourly is acceptable)

