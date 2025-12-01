---
tags:
  - workflow
  - diagram
  - sequence
  - robot
  - customer
  - delivery
---
# Customer Unlocks Robot for Pickup Flow

sequenceDiagram
    actor Customer
    participant PartnerApp as "Uber Eats / DoorDash App"
    participant PartnerBackend as "Partner Backend"
    participant IntTS as "Integrations Service (TS)"
    participant DeliveriesTS as "Deliveries Service (TS)"
    participant StateTS as State Service (TS)
    participant DeviceTS as Device Service (TS)
    participant RobotGO as Robot Service (Go)

    Customer->>+PartnerApp: 1. Clicks "Unlock Robot"
    PartnerApp->>+PartnerBackend: 2. Sends unlock request
    PartnerBackend->>+IntTS: 3. POST /integrations/v3/delivery/unlock
    IntTS->>+DeliveriesTS: 4. POST /v3/delivery/unlock (with deliveryId)
    DeliveriesTS->>+StateTS: 5. POST /state/{serial}/request-transition (to: "lid_open")
    Note over StateTS: Changes the robot's state to reflect the lid opening.
    StateTS-->>-DeliveriesTS: 6. Transition successful

    DeliveriesTS->>+DeviceTS: 7. POST /devices/{serial}/command (body: "unlock_lid")
    DeviceTS->>+RobotGO: 8. gRPC: SendCommand(command="unlock_lid")
    RobotGO-->>-DeviceTS: 9. Command sent to physical robot
    DeviceTS-->>-DeliveriesTS: 10. Command acknowledged
    DeliveriesTS-->>-IntTS: 11. Unlock command successful
    IntTS-->>-PartnerBackend: 12. Unlock command successful
    PartnerBackend-->>-PartnerApp: 13. Unlock command successful

    PartnerApp-->>-Customer: 14. Displays "Robot unlocked, please collect your items."

    Note over Customer: Customer takes items and closes the lid.
    Note over RobotGO: Robot's lid sensor detects it has been closed.
    RobotGO->>+StateTS: 15. gRPC: ReportEvent(event="lid_closed")
    StateTS->>StateTS: 16. Updates robot state to "available"

### Flow Description

1.  **Customer Action:** The customer, tracking the [[Delivery]] via the **Uber Eats or DoorDash app**, is notified of the [[Robot]]'s arrival. They press the "Unlock Robot" button.
2.  **Unlock Request to Partner:** The partner app sends an unlock request to its own backend.
3.  **Partner to [[Integrations Service]]:** The partner's backend sends a `POST` request to a partner-specific endpoint on the **[[Integrations Service]]**.
4.  **Request to [[Deliveries Service]]:** The **[[Integrations Service]]** authenticates the request and forwards it to the `/v3/delivery/unlock` endpoint on the **[[Deliveries Service]]**.
5.  **State Transition:** The **[[Deliveries Service]]** first calls the **[[State Service]]** to transition the [[Robot]]'s state to `lid_open`. This prevents the [[Robot]] from being assigned any other [[Task|tasks]] while it is open.
6.  **Send Unlock Command:** The **[[Deliveries Service]]** then calls the **[[Device Service]]** to send the physical `unlock_lid` command to the [[Robot]].
7.  **gRPC Call to [[Robot]]:** The **[[Device Service]]** relays this command via a gRPC call to the **[[Robot Service]]**, which in turn sends the command to the [[Robot]], causing the cargo bay to open.
8.  **Confirmation to Customer:** The success response is propagated back through the **[[Integrations Service]]** to the partner's backend and then to the customer's app, which displays a confirmation message.
9.  **Lid Closed Event:** After the customer retrieves their items and closes the lid, the [[Robot]]'s physical sensor detects this. The **[[Robot Service]]** (or the [[Robot]]'s firmware itself) sends a `lid_closed` event to the **[[State Service]]**.
10. **Update State:** The **[[State Service]]** updates the [[Robot]]'s state back to `available`, making it ready for its next [[Delivery]].
