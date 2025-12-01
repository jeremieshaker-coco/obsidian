---
tags:
  - dispatch-engine
  - planning
  - service-method
---
# PlannerService - Create

The `create` method in the [[PlannerService]] is responsible for committing to a delivery plan and creating the necessary entities in the system.

## Functionality

- **Input**: Takes a `DeliveryRequest` object.
- **Process**:
    1.  First, it calls the [[PlannerService - Estimate]] logic to get a valid plan.
    2.  If a valid plan is generated, it proceeds to create records in the database.
    3.  It creates all the necessary [[Demand]] entities (e.g., pickup, delivery, return) via the `DemandService`.
    4.  It creates the corresponding [[Trip]] entities for pilots via the `OperationsService`.
- **Output**: Returns a `DemandQuote` on success, or a `DemandQuoteFailure` if the plan could not be created.
