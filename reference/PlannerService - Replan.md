---
tags:
  - dispatch-engine
  - planning
  - service-method
  - continuous-replanning
---
# PlannerService - Replan

The `replan` method is the core planning loop of the [[Dispatch Engine]] and a key part of our [[Continuous Replanning]] strategy. It runs on a recurring interval and is responsible for keeping the state of the fleet's plans up-to-date.

See [[PlannerService - Replan Workflow]] for a detailed sequence diagram.

## Core Responsibilities

1.  **State Synchronization**: Fetches the latest state of all [[Robot]]s, pilots, and active [[Demand]]s.
2.  **Orphan Handling**: Detects and reassigns "orphaned" demands where a robot has become unavailable.
3.  **Estimate Generation**: Generates fresh ETAs for all scheduled demands.
4.  **State Updates**: Persists updated plans to the `DemandService` and `ResourceService`.
5.  **Demand Activation**: Activates the next scheduled demand for idle robots.
6.  **Event Publishing**: Publishes key events like ETA updates and `AtOrigin` notifications.
