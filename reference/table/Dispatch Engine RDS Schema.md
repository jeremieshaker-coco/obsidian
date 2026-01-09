---
tags:
  - database
  - prisma
  - dispatch-engine
  - backend
---
# Dispatch Engine RDS Schema

**Database**: PostgreSQL  
**Service**: [[Dispatch Engine]]  
**Schema File**: [`service/dispatch-engine/prisma/schema.prisma`](../../../delivery-platform/service/dispatch-engine/prisma/schema.prisma)

The Dispatch Engine database manages resource planning, demand scheduling, and availability estimates for robot deliveries. It maintains the current state of robots and pilots for scheduling decisions.

## Core Domain Tables

### Resource Management
- [[Robot Planning Table]] - Robot availability and scheduling state
- [[Resource Table]] - Generic resource registry (pilots and robots)

### Demand Planning
- [[Demand Table]] - Delivery/movement demand requests
- [[Plan Table]] - Execution plans for demands
- [[RobotPlan Table]] - Robot-specific plan details
- [[PilotPlan Table]] - Pilot-specific plan details

### Analytics
- [[Analytics_EstimateRequest Table]] - Estimate request tracking
- [[Analytics_EstimateResponse Table]] - Estimate response tracking
- [[Analytics_Estimate Table]] - Estimate details

### Geography
- [[Location Dispatch Table]] - Location records for dispatch

## Schema Diagram

```mermaid
erDiagram
    Robot ||--o| Demand : "has active demand"
    Robot ||--o| Demand : "has scheduled pickup"
    Robot ||--o| Demand : "has scheduled delivery"
    Robot ||--o| Demand : "has scheduled movement"
    Demand ||--|| Plan : "has plan"
    Demand ||--o{ Demand : "originates from"
    Plan }o--o| RobotPlan : "robot-specific"
    Plan }o--o| PilotPlan : "pilot-specific"
    Analytics_EstimateResponse ||--o| Analytics_Estimate : "contains"
    Analytics_EstimateRequest }o--|| Location : "origin"
    Analytics_EstimateRequest }o--|| Location : "destination"
```

## Key Enums

- [[DemandType Enum]] - Delivery, Deployment, Return, Pickup
- [[DemandStatus Enum]] - Requested, Scheduled, Pending, Active, Fulfilled, Canceled, Superseded
- [[LimitingFactor Enum]] - PilotAvailability, RobotAvailability, Unroutable, etc.
- [[ResourceType Enum]] - Pilot, Robot

## Robot State Tracking

The [[Robot Planning Table]] maintains critical scheduling state:

```mermaid
graph TD
    Robot[Robot] --> AD[Active Demand]
    Robot --> SP[Scheduled Pickup]
    Robot --> SD[Scheduled Delivery]
    Robot --> SM[Scheduled Movement]
    AD --> ETD1[ETD/ETA/UpdatedAt]
    SP --> ETD2[ETD/ETA/UpdatedAt]
    SD --> ETD3[ETD/ETA/UpdatedAt]
    SM --> ETD4[ETD/ETA/UpdatedAt]
```

Each robot can have:
- 1 active demand (currently executing)
- 1 scheduled pickup (next pickup to perform)
- 1 scheduled delivery (next delivery to perform)
- 1 scheduled movement (next repositioning move)

## Demand Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Requested: Quote requested
    Requested --> Scheduled: Plan created
    Scheduled --> Pending: Robot assigned
    Pending --> Active: Execution started
    Active --> Fulfilled: Completed
    Requested --> Canceled: Quote rejected
    Scheduled --> Canceled: Plan cancelled
    Pending --> Canceled: Assignment cancelled
    Scheduled --> Superseded: Better plan found
    Fulfilled --> [*]
    Canceled --> [*]
    Superseded --> [*]
```

## Related Concepts

- [[Dispatch Engine]] - Service using this database
- [[Demand]] - Core demand concept
- [[Supply]] - Resource availability concept
- [[Continuous Replanning]] - How plans are updated
- [[Point-to-Point (P2P) Dispatch]] - P2P dispatch feature
- [[PlannerService]] - Planning service component

