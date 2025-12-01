---
tags:
  - service
  - go
  - backend
---
# Operations Service

*   **Responsibilities:** The source of truth for what [[Trip|Trips]] exist in the system. A [[Trip]] is a generic concept of moving a [[Robot]] from point A to B, and is not aware of the specific [[Delivery]] it might be fulfilling. Manages human operators (Pilots, [[FO (Field Operator)|Field Ops]]) and their assignments to [[Trip|trips]].
*   **AWS Integration:**
    *   **RDS:** Stores operator, [[Trip]], and hub information in the `deliveryplatform` database.
