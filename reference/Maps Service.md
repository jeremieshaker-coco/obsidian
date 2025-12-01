---
tags:
  - service
  - go
  - backend
---
# Maps Service

*   **Responsibilities:** Provides mapping, routing, and geo-location functionalities.
*   **AWS Integration:**
    *   **RDS:** Stores map and location data in the `maps` PostgreSQL database. It loads this data into an in-memory geo-index for fast routing calculations.
