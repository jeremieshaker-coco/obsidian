---
tags:
  - state-machine
  - diagram
  - robot
---
# Robot State State Machine

This diagram models the operational status of a physical [[Robot]] in the fleet.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> OFF_DUTY: Robot is in storage/not in service
    OFF_DUTY --> PARKED: Field Op deploys robot to a parking lot
    PARKED --> ON_TRIP: Dispatch assigns a delivery trip
    PARKED --> OFF_DUTY: Field Op returns robot to storage
    ON_TRIP --> PARKED: Trip completed successfully
    ON_TRIP --> GROUNDED: Pilot or system reports an unrecoverable issue
    GROUNDED --> PARKED: Field Op resolves issue on-site
    GROUNDED --> OFF_DUTY: Field Op retrieves robot for major repair/storage
```

### Description of States

*   **`OFF_DUTY`**: The [[Robot]] is not in service. It is likely in a hub or storage container, not available for any [[Task|tasks]].
*   **`PARKED`**: The [[Robot]] is deployed in the field at a designated parking spot, fully operational, and is idle, waiting to be assigned a [[Trip]].
*   **`ON_TRIP`**: The [[Robot]] has been assigned a [[Trip]] (like a [[Delivery]]) and is actively working on it.
*   **`GROUNDED`**: The [[Robot]] has encountered an issue that requires physical intervention from a [[FO (Field Operator)|Field Operator]] (e.g., it's stuck or has a hardware failure). It cannot proceed with its [[Task]] until it is resolved.
