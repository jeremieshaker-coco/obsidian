---
tags:
  - dispatch-engine
  - problem-state
  - troubleshooting
---
# Dispatch Engine - Orphaned Demands

An **Orphaned Demand** in the [[Dispatch Engine]] is a [[Demand]] that the [[PlannerService]] considers valid and needing fulfillment, but which is not currently assigned to a valid/usable [[Robot]].

There are two distinct types of orphans:

## 1. Reassignment Orphans (Standard)
This is a normal part of the [[Dispatch Robot Reassignment]] process.
*   **Cause**: A assigned [[Robot]] goes offline, becomes unhealthy, or reports low battery.
*   **Detection**: The `replan` loop checks robot health.
*   **Resolution**: The system unassigns the bad robot and searches for a new candidate.

## 2. Orphaned Active Demands (Anomalous)
This is a broken state that requires code intervention or manual fixing.
*   **Definition**: The [[Demand]] status is `Active` in the database, but the assigned [[Robot]] does **not** have this demand ID in its `activeDemandId` column (it might be null, or in `scheduledDeliveryId`).
*   **Symptoms**:
    *   The [[PlannerService - Replan]] loop logs: `[scheduledDemandRequests] orphaned active demand - ignoring`.
    *   The demand is stuck. The Planner ignores it (assuming it's already running), but no robot is actually serving it.
*   **Causes**:
    *   **Race Conditions/Atomicity Failures**: e.g., In [[Dispatch Engine - handleDeliveryLoaded]], if the robot is assigned (`scheduledDeliveryId` set) but the demand status update fails or aborts, the DB says `Active` while the Robot says "I have a scheduled delivery".
    *   **Data Corruption**: Bugs in `updateRobotDemand` setting timestamps to `NULL`.
*   **Prevention**:
    *   Ensure atomic or resilient transitions in flows like `handleDeliveryLoaded`.
    *   Maintain data consistency (see [[Dispatch Engine - Data Consistency]]).

