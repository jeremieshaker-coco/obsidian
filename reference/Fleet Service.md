---
tags:
  - service
  - go
  - backend
---
# Fleet Service

*   **Responsibilities:** Manages the fleet of [[Robot|robots]] and human operators.
*   **AWS Integration:**
    *   **DynamoDB:** Uses a suite of tables including `fleet-robots`, `fleet-provider-accounts`, and `fleet-robot-heartbeats`.
    *   **ElastiCache:** Connects to the `fleet-redis` cluster for caching frequently accessed fleet data.
