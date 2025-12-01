---
tags:
---
# Lid State Resynchronization

A mechanism to resolve inconsistencies between a robot's physical lid state and its state recorded in the backend's [[Lid Cycle]].

## Problem
The backend's lid cycle state could become stale or incorrect after events like a battery swap or a pin-to-unlock action. This would prevent the robot from proceeding with deliveries, as the system would not register it as correctly loaded or unloaded.

## Solution
The [[State Service]], specifically within its `IotStreamerService`, consumes robot heartbeat data from the [[Device Service]]. This heartbeat contains the robot's actual physical lid state (`OPEN` or `CLOSED`).

The service compares the heartbeat's lid state with the currently active [[Lid Cycle]] in the database.
- If the heartbeat reports `CLOSED` but a lid cycle is `OPEN`, the service closes the existing cycle.
- If the heartbeat reports `OPEN` but no lid cycle is active, the service creates a new one.

These corrective events are recorded in the `LidCycleHistory` with a `reason` of `HEARTBEAT_RESYNC` to distinguish them from normal operations. This ensures the robot's state is accurate, preventing operational deadlocks.

See also: [[Robot Lid State State Machine]]
