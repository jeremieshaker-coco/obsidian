---
tags:
  - dispatch-engine
  - observability
  - planning
---
# Dispatch Robot Reassignment

**Robot Reassignment** is a critical process within the [[Dispatch Engine]], particularly in the [[Point-to-Point (P2P) Dispatch]] model. It occurs when a [[Robot]] that has been assigned to a [[Demand]] becomes unavailable *before* the [[Trip]] begins. The system must then find a suitable replacement to avoid a delivery failure.

This event is a key indicator of fleet and system health. Tracking it is a cornerstone of our [[Dispatch Observability]] strategy.

### The Reassignment Process

1.  **Detection**: During a [[PlannerService - Replan]], the system identifies that an assigned [[Robot]] is no longer usable for its scheduled [[Demand]] (e.g., it has gone offline, is unhealthy, or its battery is too low). This [[Demand]] is now considered "orphaned."

    *Note: This standard process is distinct from the anomalous state of [[Dispatch Engine - Orphaned Demands|Orphaned Active Demands]], where a mismatch exists between the Demand status and the Robot's assignment slot.*

2.  **Orphaning**: The system formally orphans the [[Demand]], logging the specific `issues` that caused the [[Robot]] to be dropped. The [[Demand]] is now unassigned.
3.  **Re-planning**: The orphaned [[Demand]] is put back into the planning pool. The [[Dispatch Engine]] attempts to find the next best available [[Robot]].
4.  **Logging**: If a new [[Robot]] is found, a `Robot reassignment` log event is created, capturing the `oldDeviceId`, the `newDeviceId`, and the `demandId`.

This entire lifecycle can be traced in [[Datadog Observability for Dispatch]]. A single `demandId` can be reassigned multiple times, and each event is logged, providing a clear audit trail.

Frequent reassignments can point to several underlying [[P2P Dispatch Risks]], such as network instability or endemic hardware issues.
