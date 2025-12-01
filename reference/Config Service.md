---
tags:
  - service
  - go
  - backend
---
# Config Service

*   **Responsibilities:** A critical service that provides centralized configuration management and is used by almost every other service. It keeps track of core entities like merchants, how to handle their integration-specific requests, and setup information.
*   **AWS Integration:**
    *   **RDS:** Stores configuration data in the `config` PostgreSQL database.
*   **Data Warehouse:**
    *   Configuration data synced to [[Redshift Data Warehouse]] via Fivetran
    *   Schema: [[Config RDS Schema]]
    *   Includes merchant configs, robot locations, integrations, and audit history
