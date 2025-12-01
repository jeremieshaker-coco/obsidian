---
tags:
  - workflow
  - diagram
  - sequence
  - pilot
  - delivery
  - robot
---
# Pilot Conducts a Delivery Flow

sequenceDiagram
    actor Pilot
    participant PilotUI as Pilot UI
    participant DeviceTS as Device Service (TS)
    participant RobotGO as Robot Service (Go)
    participant MapsGO as Maps Service (Go)

    Pilot->>+PilotUI: 1. Opens the assigned delivery task
    PilotUI->>+DeviceTS: 2. GET /drive-u/videopipeline/{vin}
    Note right of DeviceTS: (Establishes a persistent connection for video stream)
    DeviceTS->>+RobotGO: 3. gRPC: GetVideoStream(robotId)
    RobotGO-->>-DeviceTS: 4. Returns video stream
    DeviceTS-->>-PilotUI: 5. Forwards video stream

    loop Real-time Interaction
        PilotUI->>+DeviceTS: 6. GET /devices/state/latest?serials={robotSerial}
        DeviceTS->>+RobotGO: 7. gRPC: GetRobotState(robotId)
        RobotGO-->>-DeviceTS: 8. Returns robot state (speed, battery, etc.)
        DeviceTS-->>-PilotUI: 9. Forwards robot state

        PilotUI->>+MapsGO: 10. GET /route?location={robotCoords}
        MapsGO-->>-PilotUI: 11. Returns map tiles and route overlay

        Pilot->>PilotUI: 12. Issues a driving command (e.g., turn left)
        PilotUI->>DeviceTS: 13. POST /devices/{serial}/command (body: "turn_left")
        DeviceTS->>RobotGO: 14. gRPC: SendCommand(command="turn_left")
        RobotGO-->>DeviceTS: 15. Command sent to robot
        DeviceTS-->>PilotUI: 16. Command acknowledged
    end

    PilotUI-->>-Pilot: 17. Displays video feed, robot status, and map

### Flow Description

1.  **Open [[Task]]:** The pilot opens their assigned [[Delivery]] [[Task]] in the [[Pilot UI]].
2.  **Get Video Stream:** The [[Pilot UI]] requests the [[Robot]]'s video stream from the **[[Device Service]]**. The **[[Device Service]]** establishes a connection with the **[[Robot Service]]** (Go) to get the live feed, which is then forwarded to the UI.
3.  **Real-time Updates (Loop):** The UI enters a loop to provide the pilot with real-time information:
    *   **[[Robot]] State:** It periodically fetches the [[Robot]]'s state (speed, battery, etc.) from the **[[Device Service]]**, which gets this data from the **[[Robot Service]]**.
    *   **Map Data:** It sends the [[Robot]]'s current coordinates to the **[[Maps Service]]** to get updated map tiles and the [[Delivery]] route.
4.  **Pilot Commands:** The pilot issues driving commands through the UI (e.g., using a joystick or keyboard). These commands are sent as `POST` requests to the **[[Device Service]]**.
5.  **Send Command to [[Robot]]:** The **[[Device Service]]** translates the HTTP request into a gRPC `SendCommand` call to the **[[Robot Service]]**, which then relays the command to the physical [[Robot]] for execution.
6.  **Display Information:** The [[Pilot UI]] continuously updates to show the pilot the live video feed, the [[Robot]]'s telemetry data, and its position on the map.
