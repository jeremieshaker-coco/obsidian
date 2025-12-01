---
tags:
  - dispatch-engine
  - planning
  - service-method
---
# PlannerService - Estimate

The `estimate` method in the [[PlannerService]] is responsible for providing a price and time quote for a potential delivery. This is a read-only operation that does not persist any data.

## Functionality

- **Input**: Takes a `DeliveryRequest` object, which includes origin, destination, and timing constraints.
- **Process**:
    1.  Calculates the route using the [[Maps Service]].
    2.  Finds the best available [[Robot]] for the job by querying its internal `Planner` library, which has the latest [[Supply]] state.
    3.  Generates timing estimates based on robot availability, route travel time, and pilot availability.
- **Output**: Returns a `DeliveryEstimate` with proposed timings or a `DeliveryEstimateFailure` if the delivery cannot be fulfilled.
