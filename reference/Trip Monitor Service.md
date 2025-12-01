---
tags:
  - service
  - go
  - backend
---
# Trip Monitor Service

*   **Responsibilities:** Monitors active [[Trip|trips]] for route deviations and progress.
*   **AWS Integration:**
    *   **RDS:** Connects to its own PostgreSQL database.
    *   **ElastiCache:** Uses Redis for caching [[Trip]] data.
    *   **RabbitMQ:** Listens for [[Trip]]-related events.
