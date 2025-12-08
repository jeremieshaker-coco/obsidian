# Log-Driven Business Analytics: Proposal

## Executive Summary

**Proposal**: Make structured logs the single source of truth for both real-time monitoring (Datadog) and business analytics (Redshift).

By routing high-value "event" logs to our data warehouse, we can:
1.  **Slash Incident Resolution Time**: From 30+ minutes of manual searching to seconds of SQL querying.
2.  **Unify Metrics**: Ensure the numbers we report to the business match exactly what engineers see in the logs.
3.  **Enable "Event Replay"**: Instantly retrieve the full chronological story of any delivery, robot, or order.

---

## The Problem

**We have a disconnected view of our business.**

1.  **Logs** tell us *what happened* (in Datadog), but are hard to analyze in aggregate.
2.  **Analytical Metrics** tell us *how we're performing* (in Sigma/Redshift), but are disconnected from the logs.

**The Pain Point**: When a key metric drops (e.g., "Supply Hours are down"), we cannot easily explain *why*.
Engineers have to manually hunt through Datadog logs, searching for individual delivery IDs, trying to reconstruct the timeline of what went wrong. This is slow, manual, and prone to error.

---

## Business Impact

### 1. Drastically Reduced Time-to-Diagnosis
**Current State**: A merchant reports a failed delivery. An engineer spends 20 minutes searching Datadog for the robot ID, then the order ID, then scrolling through thousands of logs to find the error.
**Future State**: A single SQL query or Dashboard lookup returns the **Delivery Session**: a clean, chronological list of every important event that occurred for that order.
**Impact**: Incident diagnosis drops from ~20 minutes to <1 minute.

### 2. Accurate "Usable Supply Hours"
This is a critical business metric currently calculated from database snapshots, which often miss transient states. By calculating this directly from the stream of `robot.state_changed` events:
*   We capture every second of availability.
*   We can instantly drill down into *why* a robot wasn't available (e.g., "75% of downtime was due to 'Lid Error'").

### 3. Removing the "Manual Labeling" Bottleneck
To analyze complex trends (e.g., "Why are deliveries failing?"), we currently have to manually read and "label" random samples of logs.
With this change, we can query 100% of our data instantly. We can ask: *"Show me all deliveries where a robot error occurred within 2 minutes of arrival."*

### 4. Other Key Metrics
*   **Delivery Funnel**: Exact timing for Order → Dispatch → Pickup → Dropoff.
*   **Robot Reliability**: Mean Time Between Failures (MTBF) derived directly from error logs.

---

## The Solution: "Analytical Metrics" from Logs

We are **not** just piping all logs to Redshift. We are building a structured **Event Pipeline**.

### 1. The "Event Registry"
We will define a standardized set of business events in code (e.g., `Events.DELIVERY_PICKUP_COMPLETED`, `Events.ROBOT_ERROR`).
We will also introduce a dedicated `logger.event()` method to ensure these are logged consistently.

This ensures that "what we log" is exactly "what we measure." No more free-form strings.

**Example**:
```typescript
// ❌ Before: Free-form, hard to query
logger.info('Robot finished pickup', { robotId: '123' });

// ✅ After: Structured, Type-Safe, Discoverable
logger.event(Events.DELIVERY_PICKUP_COMPLETED, {
  robot_id: '123',
  delivery_id: 'D-99'
});
```

### 2. Constructing the "Delivery Session"
Raw logs are often messy. A robot log might have a `robot_id` but no `delivery_id`.
We will build data models in Redshift that **join** these disparate streams together.
*   **Goal**: You plug in a `delivery_id`, and the system retrieves the robot logs, the dispatch logs, and the merchant logs for that specific timeframe.
*   **Result**: A complete, reconstructed timeline for any business entity.

### 3. Unified Observability
This improves Datadog too. By using the Event Registry, our real-time dashboards become robust. We can build "Release Health" monitors that track the exact same events used for long-term business reporting.

---

## Success Criteria

We are not focused on just rebuilding existing charts. The goal is **Visibility**.

*   **Primary Goal**: An engineer or PM can enter a Delivery ID into a dashboard and see the full list of events for that delivery (Robot, Dispatch, Pilot) in one view.
*   **Secondary Goal**: We can calculate "Usable Supply Hours" purely from these logs, matching or exceeding the accuracy of our current method.
*   **Operational Goal**: "Time to Diagnosis" for standard delivery issues is reduced by >90%.

## Timeline

**Timeline**:
*   **Phase 1 (2 weeks)**: Infrastructure (Vector, S3, Redshift). 
*   **Phase 2 (2 weeks)**: "Event Replay" Proof-of-Concept. Focus on getting the **Delivery Session** view working for one service.
*   **Phase 3 (Ongoing)**: Expand Event Registry and migrate key metrics.

## Decision Needed
Approval to proceed with **Phase 1** (Infrastructure Setup).
