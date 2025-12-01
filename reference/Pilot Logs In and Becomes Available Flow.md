---
tags:
  - workflow
  - diagram
  - sequence
  - pilot
  - authentication
---
# Pilot Logs In and Becomes Available Flow

sequenceDiagram
    actor Pilot
    participant PilotUI as Pilot UI
    participant Auth as External Auth Provider
    participant OpsTS as Operations Service (TS)
    participant OpsDB as Ops DB (PostgreSQL)
    participant Redis
    participant RabbitMQ

    Pilot->>+PilotUI: 1. Logs in via UI
    PilotUI->>+Auth: 2. Authenticates with credentials
    Auth-->>-PilotUI: 3. Returns JWT

    PilotUI->>+OpsTS: 4. POST /api/beta/operators/sync (with JWT)
    OpsTS->>+OpsDB: 5. UPSERT INTO operators (syncs profile)
    OpsDB-->>-OpsTS: 6. Confirms operator exists
    OpsTS-->>-PilotUI: 7. 200 OK

    loop Every few seconds
        PilotUI->>+OpsTS: 8. POST /api/v1/operators/{id}/heartbeat (sends heartbeat)
        OpsTS->>+Redis: 9. SET pilot_heartbeat:{id} (updates timestamp)
        Redis-->>-OpsTS: 10. OK
        OpsTS->>+OpsDB: 11. UPDATE operators SET status='AVAILABLE'
        OpsDB-->>-OpsTS: 12. OK
        OpsTS->RabbitMQ: 13. PUBLISH "pilot.heartbeat" event
    end
    OpsTS-->>-PilotUI: 14. 200 OK

### Flow Description

1.  **Authentication:** The pilot logs in through the **[[Pilot UI]]**. The UI communicates with an **External Auth Provider** (like Auth0) to verify the pilot's credentials. On success, the provider returns a JWT.
2.  **Profile Sync:** The **[[Pilot UI]]** then calls `POST /api/beta/operators/sync` on the **[[Operations Service]]**, sending the JWT. The **[[Operations Service]]** uses this to synchronize the pilot's basic profile information (ID, name, email) from the auth provider into its own **OpsDB (PostgreSQL)**, ensuring a user record exists.
3.  **Heartbeat:** The UI begins sending a regular heartbeat to the `POST /api/v1/operators/{id}/heartbeat` endpoint on the **[[Operations Service]]**. This heartbeat signals that the pilot is online and available.
4.  **State Update & Event Publishing:** Upon receiving a heartbeat, the **[[Operations Service]]** performs three critical actions:
    *   It updates a timestamp in **Redis** to track the last heartbeat time.
    *   It updates the pilot's status to `AVAILABLE` in its **PostgreSQL database**.
    *   It publishes a `pilot.heartbeat` event to **RabbitMQ**. Other services, like the **[[Dispatch Engine]]**, listen for these events to maintain an up-to-date cache of available pilots.

This heartbeat mechanism is the core of the pilot availability system, combining a persistent state in PostgreSQL with a fast-access cache in Redis and real-time eventing via RabbitMQ.
