---
tags:
  - dispatch-engine
  - trips
  - architecture
---
# Trip Creation

To improve system reliability and predictability, all [[Trip]] creation logic is being centralized within the [[Dispatch Engine]].

Historically, different types of [[Trip|trips]] were created by different services. For example, the [[Operations Service]] might create return or deployment [[Trip|trips]]. This decentralized approach could lead to inconsistencies and race conditions where dispatch and operations services fall out of sync, negatively impacting estimates and causing incorrect cancellations.

## Centralized Model

By moving all [[Trip]] creation into the [[Dispatch Engine]], the service can act as the single source of truth for a [[Robot]]'s schedule. This includes:
- **Delivery Trips**: Created when a [[Demand]] is confirmed.
- **Deployment Trips**: For moving a [[Robot]] from a storage location to a service area.
- **Return Trips**: For returning a [[Robot]] to a storage location or merchant.

## Benefits

- **Consistency**: Guarantees that all [[Trip|trips]] are considered by the planner, preventing conflicts where a deployment [[Trip]] might disrupt a planned delivery.
- **Better API**: The [[Dispatch Engine]] can expose a cleaner, more reliable API for creating [[Trip|trips]] with certain guarantees (e.g., ensuring a new [[Trip]] doesn't conflict with existing deliveries).
- **Enables [[Point-to-Point (P2P) Dispatch]]**: Centralized [[Trip]] creation is a prerequisite for features like P2P, which needs to be able to interrupt and re-route positioning [[Trip|trips]] (deployments/returns) for high-priority deliveries.
