# Event Logging: Technical Design

> **Related**: [Event Logging - Requirements](../requirements/Event%20Logging%20-%20Requirements.md)

## Architecture Overview

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

3. **Redshift Ingestion** (new)
   - Table: `raw_events` with SUPER column for JSON
   - Scheduled COPY job (Lambda or dbt) runs hourly

4. **dbt Models** (modified)
   - Existing models refactored to query `raw_events`
   - New sessionization models for delivery timeline reconstruction

### Smart Filtering

We do **not** ingest all logs into Redshift:
- **Problem**: Terabytes of debug/info logs would be slow and expensive
- **Solution**: Vector agent filters specifically for the `event` property
- **Result**: Only high-value "business signal" events are stored; debug logs stay in Datadog

---

## TypeScript Implementation

### Event Registry

**Location**: `delivery-platform/lib/common/src/events/registry.ts`

```typescript
export const Events = {
  // --- Demand Inbound (The "Offer Funnel") ---
  OFFER_RECEIVED: 'offer.received',
  OFFER_ACCEPTED: 'offer.accepted',
  OFFER_REJECTED: 'offer.rejected',
  OFFER_EXPIRED: 'offer.expired',
  QUOTE_REQUESTED: 'quote.requested',
  QUOTE_PROVIDED: 'quote.provided',

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

  // --- Trip Execution ---
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

  // --- Delivery Execution ---
  DELIVERY_PICKUP_ARRIVAL: 'delivery.pickup_arrival',
  DELIVERY_PICKUP_STARTED: 'delivery.pickup_started',
  DELIVERY_PICKUP_COMPLETED: 'delivery.pickup_completed',
  DELIVERY_DROPOFF_ARRIVAL: 'delivery.dropoff_arrival',
  DELIVERY_DROPOFF_STARTED: 'delivery.dropoff_started',
  DELIVERY_DROPOFF_COMPLETED: 'delivery.dropoff_completed',
  DELIVERY_FAILED: 'delivery.failed',

  // --- Attempt Lifecycle ---
  ATTEMPT_STARTED: 'attempt.started',
  ATTEMPT_COMPLETED: 'attempt.completed',
  ATTEMPT_FAILED: 'attempt.failed',
  ATTEMPT_RESCUED: 'attempt.rescued',
} as const;

export type EventType = typeof Events[keyof typeof Events];
```

### Type-Safe Payloads

```typescript
// Base payload (default - all optional)
interface BaseEventPayload {
  timestamp?: string;
  source?: 'uber' | 'doordash' | 'wolt' | 'direct';
  robot_id?: string;
  delivery_id?: string;
  demand_id?: string;
  offer_id?: string;
  merchant_id?: string;
  pilot_id?: string;
  parking_lot_id?: string;
  zone_id?: string;
  error?: { code: string; message: string };
  [key: string]: unknown;
}

// Specific payloads with required fields
interface OfferReceivedPayload extends BaseEventPayload {
  offer_id: string;
  source: 'uber' | 'doordash' | 'wolt' | 'direct';
  merchant_id: string;
}

interface OfferRejectedPayload extends BaseEventPayload {
  offer_id: string;
  source: 'uber' | 'doordash' | 'wolt' | 'direct';
  rejection_reason: string;
}

interface DeliveryFailedPayload extends BaseEventPayload {
  delivery_id: string;
  error: { code: string; message: string };
}

// Event → Payload mapping
interface EventPayloadMap {
  'offer.received': OfferReceivedPayload;
  'offer.rejected': OfferRejectedPayload;
  'offer.accepted': OfferAcceptedPayload;
  'delivery.failed': DeliveryFailedPayload;
  // Events not listed use BaseEventPayload
}

// Conditional type resolution
type PayloadFor<E extends EventType> = 
  E extends keyof EventPayloadMap 
    ? EventPayloadMap[E] 
    : BaseEventPayload;

// Logger interface
interface EventLogger {
  logEvent<E extends EventType>(event: E, payload: PayloadFor<E>): void;
}
```

### Usage Examples

```typescript
// ✅ Compiles - required fields present
logger.logEvent(Events.OFFER_REJECTED, {
  offer_id: 'O-123',
  source: 'uber',
  rejection_reason: 'no_available_robots',
});

// ❌ Compile Error - missing rejection_reason
logger.logEvent(Events.OFFER_REJECTED, {
  offer_id: 'O-123',
  source: 'uber',
});

// ✅ Compiles - uses default BaseEventPayload (all optional)
logger.logEvent(Events.ROBOT_STATE_CHANGED, {
  robot_id: 'R-99',
});

// Error event (auto-logs at ERROR level)
logger.logEvent(Events.DELIVERY_FAILED, {
  delivery_id: 'D-101',
  error: { code: 'HARDWARE_FAILURE', message: 'Lid stuck' },
});
```

### Go Implementation

```go
// Manual for now, follows same naming convention
log.LogEvent(ctx, "robot.state_changed", map[string]interface{}{
  "robot_id": robotID,
  "from_state": "idle",
  "to_state": "delivering",
})
```

---

## Infrastructure

### Vector Configuration

**Deploy as Helm chart** in `coco-infra/environments/{env}/core/{region}/eks/`:

```hcl
# vector.tf
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
    condition: exists(.event)

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
      max_bytes: 10000000
      timeout_secs: 300
```

### Redshift Setup

```sql
CREATE TABLE IF NOT EXISTS analytics.raw_events (
  event_data SUPER,
  s3_file_key VARCHAR(512),
  loaded_at TIMESTAMP DEFAULT GETDATE()
)
DISTKEY (event_data.robot_id)
SORTKEY (event_data.timestamp);
```

### COPY Job

```python
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

---

## Sessionization

### Problem
When a delivery fails, we manually search Datadog logs to reconstruct what happened.

### Solution
Join events in dbt to create delivery sessions:

1. **Base Query**: Select all logs tagged with `delivery_id`
2. **Robot Join**: Join `robot_id` within assignment time windows
3. **Pilot Join**: Join `pilot_id` within assignment time windows
4. **Handle Re-assignments**: Respect time windows for Robot A → Robot B

### Example Query

```sql
WITH delivery_session AS (
  SELECT 
    event_data.event::VARCHAR as event,
    event_data.timestamp::TIMESTAMP as timestamp,
    event_data.robot_id::VARCHAR as robot_id,
    event_data.error::VARCHAR as error
  FROM analytics.raw_events
  WHERE event_data.delivery_id::VARCHAR = 'D12345'
  ORDER BY timestamp
)
SELECT * FROM delivery_session;
```

---

## Cost Breakdown

| Component | Estimated Cost | Notes |
|-----------|----------------|-------|
| **Vector** | $0 | Open source, runs on existing K8s nodes |
| **S3 Storage** | ~$23/month | 1TB/year compressed |
| **Redshift Serverless** | ~$100-150/month | Hourly queries for dbt |
| **Data Transfer** | ~$10/month | S3 → Redshift in same region |
| **Total** | **~$200/month** | |

**Comparison**: Datadog log indexing would cost $450-750/month for same volume.

---

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Vector adds latency | Low | Asynchronous; doesn't block app |
| S3 costs explode | Medium | Filter at Vector; monitor bucket size |
| COPY job fails | Medium | Alerting; idempotent (can re-run) |
| Redshift performance degrades | Low | DISTKEY/SORTKEY; materialized views |
| Wrong event names | Low | Registry with autocomplete; ESLint rules |
| TS/Go registry out of sync | Medium | Code review; future: protobuf codegen |

**Overall Risk**: Low. This is additive infrastructure—if it fails, Datadog continues working.

---

## Implementation Phases

### Phase 1: Infrastructure (2 weeks)
- [ ] Create S3 bucket with lifecycle policies
- [ ] Deploy Vector DaemonSet to dev cluster
- [ ] Create Redshift `raw_events` table
- [ ] Setup COPY job
- [ ] Validate end-to-end flow

### Phase 2: Proof of Value (1-2 weeks)
- [ ] Add `event` to robot state logs
- [ ] Build `dim_delivery_sessions` dbt model
- [ ] Create Sigma dashboard for event replay
- [ ] Validate accuracy vs. manual checking

### Phase 3: Expand (Ongoing)
- [ ] Add all delivery events to registry
- [ ] Refactor existing metrics to use `raw_events`
- [ ] Create Event Catalog documentation
- [ ] Add ESLint rule to enforce registry
- [ ] Setup TypeScript payload type checking

---

## Rollback Plan

1. Delete Vector DaemonSet (logs revert to Datadog-only)
2. Keep Redshift table but stop COPY jobs
3. Revert dbt models to query application DBs

**Data loss**: None. Logs remain in Datadog. S3 archive is additive.

---

## Integration with Existing Tools

| Tool | Change Required |
|------|-----------------|
| **Datadog** | None |
| **Sigma Computing** | Point at new Redshift tables |
| **dbt** | Add new models |
| **RabbitMQ/Events** | No change |

