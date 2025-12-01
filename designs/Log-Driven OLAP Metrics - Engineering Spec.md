# Log-Driven Business Analytics: Engineering Specification

## Objective

Unify our observability and analytics systems by making structured logs the single source of truth for both real-time monitoring (Datadog) and business analytics (Redshift).

## Current State

**What We Have**:
- ✅ Structured JSON logging
- ✅ Datadog Agent collecting all container logs
- ✅ StatsD metrics for real-time counters/timers
- ✅ Redshift data warehouse with Analytical Metrics (e.g., `robot_productivity.sql`)

**The Gap**:
- ❌ Analytical metrics are derived from analytics events, not logs
- ❌ No automated pipeline from logs → Redshift
- ❌ When metrics show anomalies, we can't trace back to the event stream
- ❌ **Root cause analysis is slow**: Requires manually reading through Datadog logs to reconstruct event sequences
- ❌ **No path for metric discovery**: We can't query historical logs to "discover" new metrics or patterns (e.g., "what % of deliveries had X happened before Y?")

## Proposed Architecture

```
┌───────────────────────────────────────┐
│  NestJS/Go Services                   │
│  (JSON logs to stdout)                │
└──────────────┬────────────────────────┘
               │
               ├─→ Datadog Agent → Datadog (unchanged)
               │   • Real-time alerting
               │   • Live log search
               │   • StatsD metrics
               │
               └─→ Vector Agent → S3 → Redshift (new)
                   • Filtered: only logs with "event" property
                   • Partitioned by date: s3://logs/dt=YYYY-MM-DD/
                   • Loaded hourly via COPY command
```

### Key Components

1. **Vector Agent** (new)
   - Deployed as DaemonSet in EKS
   - Reads container logs via Kubernetes API
   - Filters for logs containing `event` property
   - Writes to S3 in compressed JSONL format

2. **S3 Event Log Archive** (new)
   - Bucket: `coco-event-logs-{env}` 
   - Lifecycle: Transition to IA after 90 days, expire after 2 years
   - Cost: ~$23/month for 1TB/year

3. **Redshift Ingestion** (new)
   - Table: `raw_events` with SUPER column for JSON
   - Scheduled COPY job (Lambda or dbt) runs hourly
   - Loads new S3 partitions into Redshift

### Infrastructure: Smart Filtering

A key architectural decision is **Source Filtering**. We do *not* ingest all logs into Redshift.

*   **Problem**: Ingesting terabytes of debug/info logs into Redshift is slow and expensive.
*   **Solution**: The Vector agent filters specifically for the `event` property.
*   **Result**: We only pay to store high-value "business signal" events. The remaining 90% of "noise" (debug logs, stack traces) stays in Datadog for operational use but doesn't clutter our data warehouse.

### 4. dbt Models (modified)
   - Existing models like `robot_productivity.sql` refactored to query `raw_events`
   - **Sessionization Models**: New models that JOIN disparate event streams (Robot, Dispatch, Order) to create a unified "Delivery Session" view.
   - Materialized as tables for fast Sigma queries

## Code Changes Required

### Minimal: Add `event` Property to Key Logs

**Pattern**: Use events from a centralized registry (not free-form strings).

**TypeScript (NestJS)**:
```typescript
// ❌ Before - not captured for Analytics
this.logger.log('Robot state changed', {
  robot_id: robotId,
  from_state: 'idle',
  to_state: 'delivering',
});

// ✅ After - captured for Analytics, using registry
import { Events } from '@coco/common/events';

this.logger.log('Robot state changed', {
  event: Events.ROBOT_STATE_CHANGED,  // ← From registry (autocomplete, type-safe)
  robot_id: robotId,
  from_state: 'idle',
  to_state: 'delivering',
  timestamp: new Date().toISOString(),
});
```

**Go** (manual for now, follows same naming):
```go
// ❌ Before
log.Info(ctx, "Robot state changed", 
  log.WithField("robot_id", robotID),
  log.WithField("state", "delivering"),
)

// ✅ After
log.Info(ctx, "Robot state changed", 
  log.WithField("event", "robot.state_changed"),  // ← Match TS registry naming
  log.WithField("robot_id", robotID),
  log.WithField("from_state", "idle"),
  log.WithField("to_state", "delivering"),
)
```

### Event Registry: The Single Source of Truth

**Goal**: Prevent free-form strings. All events come from a centralized, documented registry.

**Location**: `delivery-platform/lib/common/src/events/registry.ts`

```typescript
/**
 * Centralized Event Registry
 * 
 * All events logged for analytics MUST be defined here.
 * This ensures stable identifiers for queries.
 * 
 * Naming convention: CATEGORY_ACTION (SCREAMING_SNAKE_CASE)
 * Value format: category.action (dot notation)
 */
export const Events = {
  // --- Robot Supply Lifecycle ---
  ROBOT_STATE_CHANGED: 'robot.state_changed',
  ROBOT_DEPLOYED: 'robot.deployed',
  ROBOT_PARKED: 'robot.parked',
  ROBOT_GROUNDED: 'robot.grounded',
  ROBOT_OFF_DUTY: 'robot.off_duty',
  
  // --- Demand (Order) Lifecycle ---
  DEMAND_CREATED: 'demand.created',
  DEMAND_ASSIGNED: 'demand.assigned',
  DEMAND_ACTIVE: 'demand.active',
  DEMAND_FULFILLED: 'demand.fulfilled',
  DEMAND_CANCELED: 'demand.canceled',
  DEMAND_ETAS_UPDATED: 'demand.etas_updated',

  // --- Trip Execution Lifecycle (The "Movement") ---
  TRIP_CREATED: 'trip.created',
  TRIP_ASSIGNED: 'trip.assigned',
  TRIP_STARTED: 'trip.started',
  TRIP_COMPLETED: 'trip.completed',
  TRIP_CANCELED: 'trip.canceled',

  // --- Pilot Lifecycle ---
  PILOT_SHIFT_STARTED: 'pilot.shift_started',
  PILOT_SHIFT_ENDED: 'pilot.shift_ended',
  PILOT_ASSIGNMENT_STARTED: 'pilot.assignment_started',
  PILOT_ASSIGNMENT_ENDED: 'pilot.assignment_ended',

  // --- Delivery Execution Lifecycle ---
  DELIVERY_PICKUP_ARRIVAL: 'delivery.pickup_arrival',
  DELIVERY_PICKUP_STARTED: 'delivery.pickup_started',
  DELIVERY_PICKUP_COMPLETED: 'delivery.pickup_completed',
  DELIVERY_DROPOFF_ARRIVAL: 'delivery.dropoff_arrival',
  DELIVERY_DROPOFF_STARTED: 'delivery.dropoff_started',
  DELIVERY_DROPOFF_COMPLETED: 'delivery.dropoff_completed',
  DELIVERY_FAILED: 'delivery.failed',
} as const;

// Type for all valid event values (for TypeScript type safety)
export type EventType = typeof Events[keyof typeof Events];

/**
 * Standard Context Interface
 * 
 * These fields should be included in event payloads whenever available
 * to enable cross-cutting queries (e.g. "Show me all errors for Merchant X").
 */
export interface AnalyticsContext {
  event: EventType;
  timestamp: string;
  
  // Core Identifiers (Include as many as available)
  robot_id?: string;
  delivery_id?: string;
  demand_id?: string;
  trip_id?: string;
  pilot_id?: string;
  merchant_id?: string; // Critical for Merchant-centric views
  
  // Error Context
  error_code?: string;
  error_message?: string;
  
  // Additional payload
  [key: string]: any;
}
```

**Usage**:
```typescript
import { Events } from '@coco/common/events';

// ✅ Good - autocomplete, type-safe, documented
logger.info('Robot state changed', {
  event: Events.ROBOT_STATE_CHANGED,
  robot_id: 'R123',
  from_state: 'idle',
  to_state: 'delivering',
});
```

### Example: The Delivery Session (Stream of Events)

The core value is being able to reconstruct the entire story of a delivery.
**Note**: Raw logs may not always have the `delivery_id` (e.g., a robot error might only have `robot_id`). We will use SQL JOINs in Redshift (dbt models) to link these events based on time windows and resource assignments.

**Hypothetical Event Stream for Delivery `D-101`**:

```json
// 1. Order Received (Demand Created)
{"ts": "12:00:00", "event": "demand.created", "demand_id": "D-101", "merchant_id": "M-55"}

// 2. Matching Engine Assigns Robot (Demand Assigned)
{"ts": "12:00:05", "event": "demand.assigned", "demand_id": "D-101", "robot_id": "R-99"}

// 3. Robot Starts Moving (Robot State Change - Linked via Robot ID + Time)
{"ts": "12:00:06", "event": "robot.state_changed", "robot_id": "R-99", "state": "moving_to_pickup"}

// 4. Trip Started (Trip Lifecycle)
{"ts": "12:00:07", "event": "trip.started", "trip_id": "T-88", "robot_id": "R-99", "merchant_id": "M-55"}

// 5. Robot Arrives (Delivery Event)
{"ts": "12:05:00", "event": "delivery.pickup_arrival", "demand_id": "D-101", "robot_id": "R-99", "merchant_id": "M-55"}

// 6. Pickup Completes (Delivery Event)
{"ts": "12:08:00", "event": "delivery.pickup_completed", "demand_id": "D-101"}

// ... transit time ...

// 7. Dropoff (Delivery Event)
{"ts": "12:20:00", "event": "delivery.dropoff_completed", "demand_id": "D-101"}

// 8. Success (Demand Fulfilled)
{"ts": "12:20:01", "event": "demand.fulfilled", "demand_id": "D-101"}
```

**Value**:
*   **SLA Tracking**: Calculate `(ts_active - ts_created)` to see how long it takes to start moving.
*   **Wait Time**: Calculate `(ts_pickup_completed - ts_pickup_arrival)` to see how long robots wait at restaurants.
*   **Root Cause**: If the stream stops after step 4, we know exactly where the failure occurred (Pickup phase).

### Event Schema Standards

**Every event log must have**:
- `event` (EventType): From the Events registry
- `timestamp` (ISO 8601): When the event occurred
- Entity ID(s): `robot_id`, `delivery_id`, `order_id`, etc. (for sessionization)

### Unified Observability: Better Datadog

The registry isn't just for Analytics. It significantly improves our operational posture in Datadog:

1.  **Reliable Dashboards**: Datadog widgets querying `event:robot.error` will never break because of a typo or log message change.
2.  **Faster Debugging**: Developers don't need to grep code to find the right log string. They look up the event name in the registry.
3.  **Release Confidence**: "Release Health" dashboards built on these stable events provide a trustworthy signal for safe deployments.

## Infrastructure: Vector Configuration

**Deploy as Helm chart** in `coco-infra/environments/{env}/core/{region}/eks/`:

```hcl
# vector.tf (simplified)
resource "helm_release" "vector" {
  namespace  = "vector"
  name       = "vector"
  chart      = "vector"
  repository = "https://helm.vector.dev"

  values = [templatefile("${path.module}/vector-values.yaml", {
    s3_bucket = "coco-event-logs-${var.environment}"
    aws_region = var.region
  })]
}
```

**vector-values.yaml**:
```yaml
sources:
  kubernetes_logs:
    type: kubernetes_logs

transforms:
  filter_events:
    type: filter
    inputs:
      - kubernetes_logs
    condition: exists(.event)  # Only logs with "event" property

sinks:
  s3:
    type: aws_s3
    inputs:
      - filter_events
    bucket: "{{ s3_bucket }}"
    region: "{{ aws_region }}"
    key_prefix: "dt=%Y-%m-%d/"
    compression: gzip
    batch:
      max_bytes: 10000000  # 10MB
      timeout_secs: 300    # 5 minutes
```

## Redshift Setup

### 1. Create Table

```sql
CREATE TABLE IF NOT EXISTS analytics.raw_events (
  event_data SUPER,  -- JSON column
  s3_file_key VARCHAR(512),
  loaded_at TIMESTAMP DEFAULT GETDATE()
)
DISTKEY (event_data.robot_id)  -- Optimize for robot queries
SORTKEY (event_data.timestamp); -- Optimize for time-range queries
```

### 2. COPY Job (Lambda or dbt)

```python
# Simplified Lambda handler
def handler(event, context):
    redshift_client.execute("""
        COPY analytics.raw_events (event_data)
        FROM 's3://coco-event-logs-prod/dt={date}/'
        IAM_ROLE 'arn:aws:iam::xxx:role/RedshiftS3Access'
        FORMAT AS JSON 'auto'
        GZIP
        DATEFORMAT 'auto';
    """.format(date=datetime.today().strftime('%Y-%m-%d')))
```

## Sessionization for Root Cause Analysis

**The Problem Today**: When a delivery fails, we have to manually search Datadog logs to reconstruct what happened.

**The Solution**: Sessionize events.
*Note*: Since not all logs have `delivery_id` (e.g. a robot hardware error), we must perform **joins** in dbt to associate a robot's events with the delivery it was assigned to at that time.

**Logic for Constructing a Session**:
1.  **Base Query**: Select all logs tagged with `deliveryId` (e.g., dispatch logs, order logs).
2.  **Robot Join**: Join with `robot_logs` where `robot_id` matches the assigned robot AND `timestamp` is within the assignment window (from `assigned` to `unassigned` events).
3.  **Pilot Join**: Join with `pilot_logs` where `pilot_id` matches the assigned pilot AND `timestamp` is within the assignment window.
4.  **Re-assignments**: The model must handle multiple assignments (e.g., Robot A → Robot B) by respecting the time windows of each assignment.

**Example Query - Failure Analysis**:
```sql
-- Get complete event stream for a failed delivery
WITH delivery_session AS (
  SELECT 
    event_data.event::VARCHAR as event,
    event_data.timestamp::TIMESTAMP as timestamp,
    event_data.robot_id::VARCHAR as robot_id,
    event_data.state::VARCHAR as state,
    event_data.error_message::VARCHAR as error_message
  FROM analytics.raw_events
  WHERE event_data.delivery_id::VARCHAR = 'D12345'
  ORDER BY timestamp
)
SELECT * FROM delivery_session;
```

**Impact**: Reduces incident diagnosis from 20-30 minutes of manual log reading to a 30-second SQL query.

## Cost Breakdown

| Component | Estimated Cost | Notes |
|-----------|----------------|-------|
| **Vector** | $0 | Open source, runs on existing K8s nodes |
| **S3 Storage** | ~$23/month | 1TB/year compressed (~5GB/day), $0.023/GB/month |
| **Redshift Serverless** | ~$100-150/month | $0.375/RPU-hour, hourly queries for dbt |
| **Data Transfer** | ~$10/month | S3 → Redshift in same region |
| **Total** | **~$200/month** | Scales with log volume |

**Cost if we used Datadog for everything**: 
- Datadog log indexing: ~$3-5/GB (vs. S3 at $0.023/GB)
- For 150GB/month: $450-750/month just for indexing
- Long-term retention: $$$

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| **Vector adds latency to logs** | Low | Vector is asynchronous; doesn't block app |
| **S3 costs explode if we log too much** | Medium | Filter at Vector (only `event` logs). Monitor S3 bucket size. |
| **COPY job fails, data missing** | Medium | Alerting on Lambda/dbt failures. Idempotent (can re-run for past dates). |
| **Redshift query performance degrades** | Low | DISTKEY/SORTKEY optimizations. Materialize views via dbt. |
| **Developers use wrong event names** | Low | Event registry with TypeScript autocomplete. ESLint rules to enforce. |
| **Event registry gets out of sync (TS vs. Go)** | Medium | Document process. Future: protobuf codegen. Short-term: code review. |
| **Breaking change to existing dashboards** | Low | Refactor queries incrementally. Keep old queries working during migration. |

**Overall Risk**: **Low**. This is additive infrastructure. If it fails, Datadog continues working.

## Workflow: The Metric Lifecycle

This architecture creates a clear promotion path for observability:

1.  **Log It**: Engineer adds structured log with `event` property (e.g., `robot.lid_stuck`).
2.  **Ad-hoc Analysis (Discovery)**:
    *   Incident occurs.
    *   We query `raw_events` in Redshift to find patterns.
    *   This allows us to **retroactively** create metrics from historical data.
3.  **Prototype**: Save the SQL query as a View or Sigma workbook.
4.  **Production Metric**: Promote the logic to a dbt model (`fct_delivery_failures`) to run on a schedule.

## Implementation Phases

See the [Implementation Plan](Log-Driven%20OLAP%20Metrics%20-%20Implementation%20Plan.md) for a detailed, task-by-task breakdown.

### Phase 1: Infrastructure (2 weeks)
- [ ] Create S3 bucket with lifecycle policies
- [ ] Deploy Vector DaemonSet to dev cluster
- [ ] Create Redshift `raw_events` table
- [ ] Setup COPY job (Lambda/dbt)
- [ ] Validate: Manually add `event` property to one log, confirm it reaches Redshift

### Phase 2: Proof of Value (1-2 weeks)
- [ ] Add `event: 'robot.state_changed'` to robot state logs
- [ ] **Build `dim_delivery_sessions` dbt model** (reconstructs delivery timelines)
- [ ] Create Sigma dashboard showing "Event Replay" for a delivery
- [ ] Validate accuracy against manual log checking

### Phase 3: Expand (Ongoing)
- [ ] Add delivery events to registry (`DELIVERY_CREATED`, `DELIVERY_TRANSITIONED`, etc.)
- [ ] Refactor `robot_productivity.sql` to use `raw_events`
- [ ] Create "Event Catalog" wiki page documenting all events
- [ ] Add ESLint rule to enforce registry usage (prevent free-form strings)
- [ ] Setup TypeScript type checking for event payloads
- [ ] Create automated alerts on session data (e.g. "Robot error rate > 5% during deliveries")

## Performance Considerations

**Query Performance**:
- Raw `raw_events` table will grow large (millions of rows/month)
- Use dbt to create materialized aggregation tables (e.g., `fct_daily_robot_stats`)
- Sigma queries hit the materialized tables, not raw logs

**Example**:
```sql
-- Don't query this in Sigma
SELECT * FROM analytics.raw_events WHERE ...;  

-- Query this instead (dbt materializes it hourly)
SELECT * FROM analytics.fct_daily_robot_stats WHERE ...;
```

## Integration with Existing Tools

| Tool | Change Required |
|------|-----------------|
| **Datadog** | None (logs still flow, StatsD unchanged) |
| **Sigma Computing** | Point dashboards at new Redshift tables |
| **dbt** | Add new models for log-based metrics |
| **RabbitMQ/Events** | No change (this is about *logs*, not message bus) |

## Rollback Plan

If we need to abort:
1. Delete Vector DaemonSet (logs revert to Datadog-only)
2. Keep Redshift table but stop COPY jobs
3. Revert dbt models to query application DBs directly

**Data loss**: None. Logs are still in Datadog. S3 archive is additive.

## Success Metrics (Engineering)

- [ ] Vector deployed to all clusters (dev, staging, prod)
- [ ] Event registry created with at least 10 documented events
- [ ] **Delivery Session reconstruction works for 99% of deliveries**
- [ ] One existing Analytical Metric refactored to use `raw_events`
- [ ] Sigma dashboard queries run in <5 seconds
- [ ] Zero P0/P1 incidents caused by Vector or Redshift ingestion
- [ ] "Event Catalog" wiki page created and maintained
- [ ] Team performs at least one "ad-hoc discovery" query instead of manual log reading

## Next Steps

1. **Approve infrastructure setup** (S3, Vector, Redshift, COPY job)
2. **Define initial event schema** (10-15 events to start with)
3. **Assign owner** for Vector/Redshift (Infrastructure team? Backend team?)
4. **Pilot in dev** (1 week validation before staging/prod)
