---
tags:
  - dispatch-engine
  - planning
  - edge-cases
---
# P2P Dispatch Scenarios

The [[Point-to-Point (P2P) Dispatch]] model must account for several distinct scenarios to effectively assign robots to deliveries. The [[P2P Dispatch Sequence Flow]] provides the framework for evaluating these cases.

### Case 1: Idle Robot (The Simple Case)
-   **Scenario:** A healthy, charged, and idle [[Robot]] is located near the merchant.
-   **Action:** The system plans a standard pickup and delivery [[Trip]].

### Case 2: Interruptible Active Robot (The Core P2P Case)
-   **Scenario:** A [[Robot]] is active but on a low-priority [[Trip]] (e.g., returning to a parking lot). A high-priority delivery [[Demand]] comes in.
-   **Action:** The system calculates if interrupting the current [[Trip]] and re-routing the [[Robot]] to the new merchant is more efficient. If so, it cancels the old [[Trip]] and creates a new one for the delivery.

### Case 3: Robot Requires a Pickup Trip
-   **Scenario:** The best [[Robot]] is available but is not at the merchant's location (e.g., it just finished a delivery nearby).
-   **Action:** The system creates a `Pickup` [[Demand]] to move the [[Robot]] from its current location to the merchant before the `Delivery` [[Demand]] can begin.

### Case 4: No Suitable Robots
-   **Scenario:** No [[Robot|robots]] are nearby, or all nearby [[Robot|robots]] are unhealthy, have low battery, or are on critical, uninterruptible [[Trip|trips]].
-   **Action:** The system rejects the quote, providing `RobotAvailability` as the limiting factor. This is a key failure mode to monitor via [[Dispatch Observability]].

### Case 5: No Available Pilots
-   **Scenario:** A perfect [[Robot]] is in position, but no [[Pilot|pilots]] are available (or will be available) for the duration of the [[Trip]].
-   **Action:** The system rejects the quote, providing `PilotAvailability` as the limiting factor.

### Case 6: Re-planning / Reassignment
-   **Scenario:** A plan has been created and a [[Robot]] assigned, but before the delivery starts, the [[Robot]] becomes unavailable.
-   **Action:** The `replan` process initiates a [[Dispatch Robot Reassignment]] to find a new [[Robot]].
