---
tags:
  - dispatch-engine
  - workflow
  - robot-loading
---
# Dispatch Engine - handleDeliveryLoaded

The `handleDeliveryLoaded` flow is a critical operational sequence in the [[Dispatch Engine]]. It is triggered (typically via API) when a physical [[Robot]] is loaded with a [[Delivery]].

## Objective
The goal is to:
1.  Assign the specific [[Delivery]] [[Demand]] to the specific [[Robot]].
2.  Transition the [[Demand]] status to `Pending` (indicating it is ready for pilot assignment/trip activation).

## Sequence

1.  **Status Check**: The system first checks the current status of the [[Demand]].
    *   If the [[Demand]] is already `Active`, the system calls `markActive` (forcefully) to ensure the [[Robot]]'s state matches the DB state (putting the demand in the `activeDemand` slot). This prevents "Active Orphans".
    *   If the [[Demand]] is `Scheduled` (or other), it proceeds to assignment.

2.  **Robot Assignment (`assignDeliveryToRobot`)**:
    *   The [[Demand]] is assigned to the [[Robot]]'s `scheduledDelivery` slot in the database.
    *   Any previous assignments in that slot are cleared.
    *   **Note**: At this exact moment, if the Demand was previously `Active` but the check in Step 1 was missed, the Demand would be `Active` in the DB but `Scheduled` on the Robot, creating an [[Dispatch Engine - Orphaned Demands|Orphaned Active Demand]].

3.  **Status Update (`markPending`)**:
    *   The [[Demand]] status is updated to `Pending`.
    *   **Resilience**: This step logs a warning but *proceeds* even if the [[Demand]] is missing a `tripId`, provided `activateTrip` is false. This ensures the physical loading action is reflected in the system state even if trip data is imperfect.

## State Transitions

*   **Standard**: `Scheduled` -> `Pending`
*   **Recovery**: `Active` -> `Active` (re-confirms assignment to `activeDemand` slot)

This flow is designed to be idempotent and resilient to partial state inconsistencies.

