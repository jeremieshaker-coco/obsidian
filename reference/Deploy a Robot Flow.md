---
tags:
  - workflow
  - diagram
  - sequence
  - robot
  - deployment
---
# Deploy a Robot Flow

sequenceDiagram
    actor User
    participant MC as Mission Control UI
    participant OpsTS as Operations Service (TS)
    participant OpsDB as Operations DB (PostgreSQL)
    participant StateTS as State Service (TS)
    participant FleetGO as Fleet Service (Go)
    participant RabbitMQ as RabbitMQ

    User->>+MC: 1. Clicks "Deploy Robot"
    MC->>+OpsTS: 2. POST /api/v2/robots/{serial}/deploy
    
    OpsTS->>+OpsDB: 3. CREATE RobotCheckInHistory (state: CHECKED_OUT)
    OpsDB-->>-OpsTS: 4. Record created
    
    OpsTS->>+FleetGO: 5. gRPC: getRobotLocations
    FleetGO-->>-OpsTS: 6. Returns location details

    OpsTS->>+OpsDB: 7. CREATE RobotDeployment
    OpsDB-->>-OpsTS: 8. Record created

    OpsTS->>+StateTS: 9. gRPC: requestTransition (to: PARKED)
    StateTS-->>-OpsTS: 10. Transition successful

    OpsTS->>+RabbitMQ: 11. PUBLISH (Robots.Deployment)
    OpsTS->>+RabbitMQ: 12. PUBLISH (Robots.StateChange: PARKED)
    
    OpsTS-->>-MC: 13. Robot deployment successful
    MC-->>-User: 14. Displays "Robot deployed"

### Flow Description

1.  **Initiate Deployment:** The user finds a grounded [[Robot]] in the [[Mission Control UI]] and clicks the "Deploy" button.
2.  **Request to [[Operations Service]]:** The [[Mission Control UI]] sends a `POST` request to the `/api/v2/robots/{serial}/deploy` endpoint on the **[[Operations Service]]**.
3.  **Log Checkout:** The **[[Operations Service]]** writes to its own **PostgreSQL database** (via Prisma) to create a `RobotCheckInHistory` record, marking the [[Robot]] as `CHECKED_OUT` of its storage location.
4.  **Get Location Details:** It then makes a gRPC call to the Go-based **[[Fleet Service]]** to get information about the deployment destination (the parking lot).
5.  **Log Deployment:** The **[[Operations Service]]** writes to its database again to create a permanent `RobotDeployment` record, logging the reason for the deployment and who triggered it.
6.  **Update State:** The **[[Operations Service]]** sends a gRPC call to the **[[State Service]]**, requesting a transition of the [[Robot]]'s state to `PARKED`.
7.  **Publish Events:** After the state is successfully updated, the **[[Operations Service]]** publishes two events to **RabbitMQ**:
    *   A `RobotDeployment` event to notify any interested services that a deployment has occurred.
    *   A `RobotStateUpdated` event to broadcast that the [[Robot]]'s state is now `PARKED`.
8.  **Confirmation to UI:** Once all backend operations are complete, the service returns a success response to the [[Mission Control UI]], which then displays a confirmation to the user.
