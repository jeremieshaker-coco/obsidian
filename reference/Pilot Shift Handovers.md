---
tags:
  - feature
  - dispatch-engine
  - pilots
---
# Pilot Shift Handovers

**Pilot Shift Handovers** is a feature of the [[Point-to-Point (P2P) Dispatch]] system that allows a single [[Trip]] to be scheduled across the shifts of multiple pilots.

## How it Works

A long-duration [[Trip]] can be started by one pilot and completed by another. For example, Pilot 1 completes the first part of a delivery before their shift ends, and then hands over control of the [[Robot]] to Pilot 2, who completes the remainder of the [[Trip]].

### Considerations

- **Handover Time**: The system needs to account for the additional time required for the handover process, which includes the first pilot parking the [[Robot]], disconnecting, and the second pilot connecting. This buffer is critical for maintaining accurate delivery estimates.
- **Pilot Schedule Accuracy**: This feature relies on accurate pilot schedules. If an expected pilot does not start their shift on time, a delivery [[Trip]] could be stalled, leading to a failure.

## Benefits

- **Increased Pilot Utilization**: This feature unlocks the ability to use fractional blocks of a pilot's time that might otherwise be wasted at the beginning or end of their shifts.
- **Fulfillment of Longer Trips**: It becomes easier to fulfill longer delivery [[Trip|trips]] that might exceed the remaining time in a single pilot's shift.
