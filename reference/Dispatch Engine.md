---
tags:
  - service
  - typescript
  - backend
---
# Dispatch Engine

The `@dispatch-engine` is a [[TypeScript Backend Services|NestJS-based microservice]] that serves as the brain for automated planning and dispatching of [[Robot|robots]] and pilots to fulfill delivery demands. Its primary goal is to optimize the assignment of resources to tasks based on various constraints like location, availability, and time.

## Core Responsibilities
- Real-time planning of resources.
- Generates time estimates for delivery offers and manages reservations for [[Robot|robots]] and pilots.
- A worker runs every 15 seconds to look at all accepted demand and update assignments and ETAs for active [[Trip|trips]].
- It is the source of truth for pilot and [[Robot]] availability.
- Continuously replanning to adapt to real-world changes.
- Orchestrating the lifecycle of a [[Demand]].

## Key Architectural Concepts
- [[Dispatch Engine Architecture]]
- [[Dispatch Engine Workflow]]
- [[Continuous Replanning]]
- [[Point-to-Point (P2P) Dispatch]]
- [[Event-Driven Architecture]]

## AWS Integration
- **ElastiCache:** Connects to the `dispatch-engine-redis` cluster for caching fleet and location data to make quick dispatch decisions.

## Related Services
- [[Operations Service]]: Manages the execution of a [[Trip]].
- [[State Service]]: Provides real-time state information for [[Robot|robots]] and pilots.
- [[Maps Service]]: Provides routing and travel time information.
