---
tags:
  - state-machine
  - diagram
  - pilot
---
# Pilot State State Machine

This diagram shows the status of a remote pilot as they move through their workday, from the perspective of the [[Dispatch Engine]].

```mermaid
stateDiagram-v2
    direction LR
    [*] --> OffShift: Pilot is not working
    OffShift --> OnShift: Pilot logs in for their shift
    OnShift --> OnBreak: Pilot takes a scheduled break
    OnShift --> OnCooldown: Pilot finishes a trip and has a brief cooldown period
    OnShift --> Unresponsive: Pilot is not responding to heartbeats
    OnShift --> OffShift: Pilot logs out at the end of shift
    OnBreak --> OnShift: Pilot's break ends
    OnCooldown --> OnShift: Cooldown period ends
    Unresponsive --> OnShift: Pilot's connection is restored
    Unresponsive --> OffShift: System marks pilot as offline after timeout
```

### Description of States

*   **`OffShift`**: The pilot is not working and cannot be assigned any [[Task|tasks]].
*   **`OnShift`**: The pilot is logged in, on shift, and is considered available to be assigned a [[Delivery]] [[Trip]].
*   **`OnBreak`**: The pilot is on a scheduled break and is temporarily unavailable for new assignments.
*   **`OnCooldown`**: After completing a [[Task]], the pilot enters a brief cooldown period before they can be assigned a new one.
*   **`Unresponsive`**: The system has not received a heartbeat from the pilot's UI for a period of time, and they are considered temporarily unavailable.
