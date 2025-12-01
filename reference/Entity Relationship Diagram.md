---
tags:
  - diagram
  - erd
  - database
  - architecture
---
# Entity Relationship Diagram

erDiagram
    Delivery ||--o{ DeliveryAttempts : "has"
    Delivery {
        string id PK
        string quoteId FK
        string latestAttemptId FK
        string partnerId
        DeliveryStatus status
    }
    Quote {
        string id PK
        string externalReferenceId
        string merchantId
        string pickupLocationId FK
        string dropoffLocationId FK
    }
    Delivery }o--|| Quote : "is for"
    Location {
        string id PK
        float lat
        float lng
        string fullAddress
    }
    Quote }o--|| Location : "pickup"
    Quote }o--|| Location : "dropoff"
    Attempt {
        string id PK
        AttemptProvider provider
        AttemptStatus status
        string providerId
    }
    DeliveryAttempts }o--|| Attempt : "tracks"
    Task {
        string id PK
        TaskStatus status
        TaskType type
        string robotSerial FK
        string deliveryId
        string attemptId
    }
    Trip {
        string id PK
        string taskId FK
        TripStatus status
        TripType type
    }
    Task ||--|{ Trip : "has"
    Robot {
        string serial PK
        string displayName
    }
    Task }o--|| Robot : "is performed by"
    Operator {
        string id PK
        string email
        OperatorStatus status
        OperatorRole role
    }
    PilotAssignmentModel {
        string id PK
        string pilotId FK
        string taskId FK
    }
    PilotAssignmentModel }o--|| Operator : "assigns to"
    PilotAssignmentModel }o--|| Task : "is for"
    StateHistory {
        string serial PK
        datetime date PK
        string state
    }
    Robot ||--o{ StateHistory : "has history of"

### Key Relationships and Insights

*   **Central Role of `[[Delivery]]`**: The `Delivery` table is central. It's linked to a `Quote` (the initial request) and has multiple `DeliveryAttempts`. This clearly models the process where a single [[Delivery]] might require multiple tries or couriers.
*   **Decoupling of `[[Delivery]]` and `[[Task]]`**: `[[Delivery]]` (in the `[[deliveries Service]]`) and `[[Task]]` (in the `[[operations Service]]`) are linked by `deliveryId` and `attemptId`. This is a great example of **loose coupling between services**. The `[[deliveries Service]]` cares about the *what* (the order), while the `[[operations Service]]` cares about the *how* (the specific [[Task|tasks]] and [[Trip|trips]] to get it done).
*   **Operational Hub (`[[Task]]` and `[[Trip]]`)**: The `[[Task]]` and `[[Trip]]` tables are the heart of the `[[operations Service]]`. A `[[Task]]` is a high-level goal (e.g., "complete this [[Delivery]]"), while a `[[Trip]]` is the specific robot journey.

## Data Warehouse Integration

This operational ERD is mirrored in the [[Redshift Data Warehouse]] for analytics:

- **Operational Tables** → [[Deliveries V3 RDS Schema]] - Current delivery operations
- **Legacy Tables** → [[Deliveries RDS Schema]] - Historical delivery data  
- **Configuration** → [[Config RDS Schema]] - Merchant and robot configurations
- **Events** → [[Delivery Platform Events Schema]] - User interactions and system events
- **CRM Data** → [[Salesforce Data Sync]] - Sales and customer data

See [[Redshift Query Patterns]] for common queries joining operational and analytical data.
