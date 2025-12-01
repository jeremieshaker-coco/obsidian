---
tags:
  - dispatch-engine
  - planning
  - sequence-diagram
  - architecture
---
# P2P Dispatch Sequence Flow

This diagram illustrates the "happy path" sequence for a [[Point-to-Point (P2P) Dispatch]] quote request, from the initial API call to the final creation of a delivery plan. This flow is the backbone of the P2P model, enabling a more dynamic and responsive dispatch system.

It shows the interactions between the [[Dispatch Engine]], the [[Deliveries Service]], and other key backend services like the [[Maps Service]] and [[Fleet Service]].

```mermaid
sequenceDiagram
    participant Client as Deliveries Service
    participant DE as Dispatch Engine
    participant Maps as Maps Service
    participant Fleet as Fleet Service
    participant Demands as Demand DB

    Client->>+DE: POST /v1/demand/quote (Request a plan)
    DE->>+Maps: Get Route(origin, destination)
    Maps-->>-DE: Route Information (time, distance)

    DE->>+Fleet: Get Available Robots(near origin)
    Fleet-->>-DE: List of Robots (location, status, active trips)

    loop For Each Robot
        DE->>DE: Evaluate Robot for Delivery
    end
    Note right of DE: Select best robot based on ETA.<br/>This includes potentially<br/>interrupting an active return trip.

    DE->>+Demands: Create & Persist Demands (Delivery, Return, etc.)
    Demands-->>-DE: Confirmed Demands

    DE-->>-Client: 201 Created (DemandQuote with plan)
```

This sequence is the foundation for handling various [[P2P Dispatch Scenarios]], but it's important to consider the [[P2P Dispatch Risks]] associated with acting on potentially stale state data from the Fleet Service. Proper [[Dispatch Observability]] is crucial for monitoring the health of this flow.
