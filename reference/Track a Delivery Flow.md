---
tags:
  - workflow
  - diagram
  - sequence
  - delivery
  - monitoring
---
# Track a Delivery Flow

sequenceDiagram
    actor User
    participant MC as Mission Control UI
    participant DeliveriesTS as Deliveries Service (TS)
    participant DeliveriesDB as Deliveries DB (PostgreSQL)
    participant DeviceTS as Device Service (TS)
    participant DeviceStateDB as Device State DB (DynamoDB)
    participant MapsGO as Maps Service (Go)

    User->>+MC: 1. Opens the delivery tracking page
    MC->>+DeliveriesTS: 2. GET /api/v2/delivery/{id}/info
    DeliveriesTS->>+DeliveriesDB: 3. SELECT * FROM deliveries WHERE id=...
    DeliveriesDB-->>-DeliveriesTS: 4. Returns delivery details (e.g., robotSerial)
    DeliveriesTS-->>-MC: 5. Returns delivery details

    loop Every few seconds
        MC->>+DeviceTS: 6. GET /api/v2/devices/gps/batch?serials={robotSerial}
        DeviceTS->>+DeviceStateDB: 7. GET last known state for robot
        DeviceStateDB-->>-DeviceTS: 8. Returns robot's GPS coordinates
        DeviceTS-->>-MC: 9. Returns robot location
    end

    MC->>+MapsGO: 10. (API Call) GetMapData(coordinates)
    MapsGO-->>-MC: 11. Returns map tiles and route
    MC-->>-User: 12. Displays robot's location on the map

### Flow Description

1.  **Open Tracking Page:** The user navigates to the tracking page for a specific [[Delivery]] in the [[Mission Control UI]].
2.  **Fetch [[Delivery]] Info:** The UI first calls the `GET /api/v2/delivery/{id}/info` endpoint on the **[[Deliveries Service]]**.
3.  **Query [[Deliveries Service|Deliveries Database]]:** The **[[Deliveries Service]]** queries its own **PostgreSQL database** to get the details of the [[Delivery]], including which [[Robot]] (`robotSerial`) is assigned to it.
4.  **Fetch [[Robot]] Location:** The UI then begins polling the `GET /api/v2/devices/gps/batch` endpoint on the **[[Device Service]]** for the [[Robot]]'s current GPS location.
5.  **Query Device State Database:** The **[[Device Service]]** does not call another service. Instead, it uses the `StateTrackerService` to read the last known state for the [[Robot]] directly from the **Device State DynamoDB table**. This table is kept up-to-date by the **[[State Service]]**, which processes incoming data from the physical [[Robot|robots]] via AWS IoT and SQS.
6.  **Fetch Map Data:** Once the UI receives the [[Robot]]'s coordinates, it makes an API call to the **[[Maps Service]]** to get the corresponding map tiles and route information.
7.  **Display on Map:** Finally, the [[Mission Control UI]] renders the [[Robot]]'s location on the map. The polling mechanism ensures the location is updated in near real-time.
