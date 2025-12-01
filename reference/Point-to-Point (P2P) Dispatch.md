---
tags:
  - feature
  - dispatch-engine
  - planning
---
# Point-to-Point (P2P) Dispatch

**Point-to-Point (P2P) Dispatch** is a major evolution of the [[Dispatch Engine]]'s planning capabilities. The primary goal is to significantly improve our delivery acceptance rate by enabling [[Robot|robots]] to perform pickups from any location, not just pre-defined parking lots.

This new model moves the [[Dispatch Engine]] from a parking-lot-based model to a more flexible position-based model.

## Key Goals & Features

- **Improve Acceptance Rate**: By being more flexible about pickup locations, we can accept more [[Demand|demands]].
- **Enable Pickups from Anywhere**:
    - This includes non-parking lot locations such as customer locations or storage facilities.
    - It also allows interrupting active positioning [[Trip|trips]] (like deployments and returns) to start a new delivery [[Trip]].
- **[[Pilot Shift Handovers]]**: Allows [[Trip|trips]] to be scheduled across pilot shifts, enabling more efficient use of pilot time.
- **Enhanced [[Dispatch Observability]]**: The system will persist more detailed data about rejection reasons and provide better visibility into [[Robot]] schedules and estimates over time.
- **Centralized [[Trip Creation]]**: All [[Trip]] creation logic (including deployment and return [[Trip|trips]]) will be consolidated within the [[Dispatch Engine]], leading to a more robust and predictable system.

## How it Works

Under P2P, the dispatch model changes fundamentally:

1.  **Position-Based Model**: Instead of planning routes from fixed parking lots, the system plans from the [[Robot]]'s current or projected position.
2.  **Dynamic Pickups**: A [[Robot]] returning to a merchant or being deployed can be dynamically re-routed to a new pickup location for a higher-priority delivery.
3.  **Simplified Scheduling**: To begin with, the new model will only allow one scheduled delivery per [[Robot]]. A [[Robot]] can only accept a new delivery if it is idle or has an in-progress delivery. This simplifies the planning logic and avoids cascading delays from complex schedules.

### Impact on [[Robot]] Behavior

- **Robot Drift**: [[Robot|Robots]] will naturally drift away from their start-of-day locations. Initially, they will be routed back to the merchant they picked up from.
- **Future Optimization**: There's an opportunity to route [[Robot|robots]] to the "best" parking lot based on historical demand data and distance, rather than just returning them to the last pickup merchant.

## Technical Updates

- The quote flow is updated to generate a delivery estimate that, if accepted, will not bump or delay previously accepted deliveries.
- The new implementation refactors previous "hacks" like implicit pickups and delayed returns, making the core dispatch logic easier to maintain.
- The `planner.service.ts` and `planner.lib.ts` in the `@dispatch-engine` service are the core components being updated for this feature.

## Metrics Impact

- **Acceptance Rate**: Expected to increase.
- **Robot Booked Time**: Expected to decrease as [[Trip|trips]] become more efficient (e.g., no unnecessary return to a parking lot before a pickup).
- **Pilot Booked Time**: Expected to decrease proportionally with [[Robot]] booked time.
- **Pilot Utilization**: Expected to increase due to the ability to use fractional blocks of pilot time across shifts.
