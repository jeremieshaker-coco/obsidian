---
tags:
  - workflow
  - diagram
  - sequence
  - maps
  - ui
---
# Editing a Service Region Flow

sequenceDiagram
    actor Operator
    participant MapsUI_Frontend as Maps UI (Browser)
    participant MapsUI_Backend as Maps UI (Next.js API Route)
    participant MapsGO as Maps Service (Go)

    Operator->>+MapsUI_Frontend: 1. Edits a service region on the map and clicks "Save"

    MapsUI_Frontend->>+MapsUI_Backend: 2. POST /api/v1/service-regions/{id} (with new coordinates)

    Note over MapsUI_Backend: 3. API Route receives the request from the browser.

    MapsUI_Backend->>+MapsGO: 4. gRPC: UpdateServiceRegion(id, coordinates)

    MapsGO-->>-MapsUI_Backend: 5. Returns success/failure

    MapsUI_Backend-->>-MapsUI_Frontend: 6. 200 OK or Error

    MapsUI_Frontend-->>-Operator: 7. Displays "Service Region Saved"

### Flow Description

1.  **User Action:** An operator in the standalone **[[Maps UI]]** application edits the boundaries of a service region and clicks "Save."
2.  **Frontend to BFF:** The browser sends a `POST` request to its own backend, an API Route living at `/api/v1/service-regions/{id}`.
3.  **BFF Receives Request:** The Next.js serverless function receives this request.
4.  **BFF to Backend:** The API Route, running on the server, then makes a secure, server-to-server gRPC call to the **[[Maps Service]]** (Go), passing along the updated coordinates for the service region.
5.  **Confirmation:** The **[[Maps Service]]** validates and saves the data, then returns a success message to the Next.js API Route.
6.  **Response to Frontend:** The API Route sends a `200 OK` response back to the browser.
7.  **Display to User:** The [[Maps UI]] in the browser receives the successful response and shows a confirmation message to the operator.
