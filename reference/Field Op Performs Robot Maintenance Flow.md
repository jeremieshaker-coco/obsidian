---
tags:
  - workflow
  - diagram
  - sequence
  - field-op
  - robot
  - maintenance
---
# Field Op Performs Robot Maintenance Flow

sequenceDiagram
    actor FieldOp as Field Operator
    participant FOUIs as Field Op UI
    participant OpsTS as Operations Service (TS)
    participant TaskGO as Task Service (Go)
    participant StateTS as State Service (TS)
    participant FleetGO as Fleet Service (Go)

    FieldOp->>+FOUIs: 1. Receives notification and opens the assigned maintenance task
    FOUIs->>+OpsTS: 2. GET /beta/tasks/{taskId}
    OpsTS-->>-FOUIs: 3. Returns task details (robot location, issue)
    
    Note over FieldOp: Travels to the robot's location and performs the battery swap.

    FieldOp->>+FOUIs: 4. Clicks "Mark Task as Complete"
    FOUIs->>+OpsTS: 5. POST /beta/tasks/{taskId}/complete
    OpsTS->>+TaskGO: 6. gRPC: Complete(taskId)
    Note over TaskGO: Marks the maintenance task as completed.
    TaskGO->>+FleetGO: 7. gRPC: UpdateProviderStatus(fieldOpId, status="available")
    Note over FleetGO: Frees up the Field Op for new tasks.
    FleetGO-->>-TaskGO: 8. Field Op status updated
    TaskGO-->>-OpsTS: 9. Task completion successful

    OpsTS->>+StateTS: 10. POST /state/{serial}/request-transition (to: "available")
    Note over StateTS: Robot is now ready for deliveries again.
    StateTS-->>-OpsTS: 11. Transition successful
    
    OpsTS-->>-FOUIs: 12. Task completion confirmed
    FOUIs-->>-FieldOp: 13. Displays "Task Complete. Robot is now available."

### Flow Description

1.  **Open [[Task]]:** The [[FO (Field Operator)|Field Operator]] receives a notification on their device (e.g., a phone or tablet) about the new high-priority maintenance [[Task]]. They open the **Field Op UI** to view the details.
2.  **Get [[Task]] Details:** The UI fetches the [[Task]] details from the **[[Operations Service]]**, which shows the [[Robot]]'s location and the nature of the issue (e.g., "Battery Swap Required").
3.  **Perform Maintenance:** The [[FO (Field Operator)|Field Op]] travels to the [[Robot]] and performs the physical [[Task]] (swaps the battery).
4.  **Complete [[Task]]:** After the work is done, the [[FO (Field Operator)|Field Op]] marks the [[Task]] as complete in their UI.
5.  **gRPC Call to [[Task Service]]:** The UI sends a request to the **[[Operations Service]]**, which in turn sends a gRPC `Complete` call to the **[[Task Service]]** to officially close out the maintenance [[Task]].
6.  **Update [[FO (Field Operator)|Field Op]] Status:** The **[[Task Service]]** then calls the **[[Fleet Service]]** to update the [[FO (Field Operator)|Field Op]]'s status back to `available`, so they can be assigned new [[Task|tasks]].
7.  **Update [[Robot]] State:** Critically, the **[[Operations Service]]** then calls the **[[State Service]]** to transition the [[Robot]]'s state from `grounded` back to `available`.
8.  **Confirmation:** The success message is propagated back to the **Field Op UI**, confirming that the [[Task]] is complete and the [[Robot]] is back in service.
