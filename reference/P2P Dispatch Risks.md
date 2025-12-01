---
tags:
  - dispatch-engine
  - planning
  - risks
---
# P2P Dispatch Risks

While the [[Point-to-Point (P2P) Dispatch]] model offers significant advantages in flexibility, it also introduces several risks that must be managed through careful implementation and robust [[Dispatch Observability]].

### 1. Stale State Data
-   **Risk:** The biggest risk in a position-based model is acting on old data. If the [[Dispatch Engine]] has a stale location, battery level, or trip status for a [[Robot]], it could generate a plan that is impossible to execute, leading to inevitable [[Dispatch Robot Reassignment|reassignments]] or fulfillment failures.
-   **Mitigation:** Ensure high-frequency, reliable [[Active Robot Heartbeat|heartbeats]] from the fleet and build resilience in the planner to handle data inconsistencies.

### 2. Race Conditions
-   **Risk:** Although the P2P design simplifies scheduling by allowing only one scheduled delivery per [[Robot]], there is still potential for two `quote` requests to evaluate the same idle [[Robot]] simultaneously. The first one to persist the [[Demand]] will win, but the second might fail unexpectedly.
-   **Mitigation:** Implement optimistic concurrency control or locking mechanisms at the demand creation step to handle simultaneous requests gracefully.

### 3. Incorrect Trip Interruption
-   **Risk:** The logic to decide whether to interrupt an active [[Trip]] is complex. A bug could cause the system to cancel a return [[Trip]] for a delivery that is only marginally more efficient, leading to poor overall fleet positioning and "robot drift."
-   **Mitigation:** Extensive simulation and testing of the interruption algorithm. Add metrics to track the frequency and "value" of interruptions to allow for post-deployment analysis.

### 4. "Orphan" Failures
-   **Risk:** If a [[Robot]] becomes unavailable after being assigned, the `replan` loop must successfully find a replacement. If it fails (as outlined in [[P2P Dispatch Scenarios]]), the delivery could be left in a "limbo" state without a [[Robot]], leading to a fulfillment failure.
-   **Mitigation:** The [[Datadog Observability for Dispatch]] we've implemented is key here. By creating metrics and alerts on failed reassignments, we can quickly identify and address issues that prevent a successful handover.
