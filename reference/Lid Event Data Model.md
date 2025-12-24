---
tags:
  - delivery
  - operations
  - robot
  - data-model
---
# Lid Event Data Model

The robot loading and unloading process involves a hierarchy of events and states that track the physical actions of the lid and the business logic surrounding them. Understanding the distinction between these layers is critical for debugging issues like missing loads or race conditions.

## Event Hierarchy

```mermaid
graph TD
    Raw[Device Events (Raw)] -->|Ingestion| Processed[Lid Events (Normalized)]
    Processed -->|Aggregation| Cycle[Lid Cycle State]
    Processed -->|Logging| History[Lid Cycle History]
```

### 1. Device Events (Raw)
These are the low-level telemetry messages received directly from the robot firmware via the IoT pipeline (SQS -> `IotStreamer`). They contain raw sensor data and flags.

*   **Source:** Robot Firmware (`coco-acu`)
*   **Examples:**
    *   `pin_entry_event` with `lid_opened: true` (The physical keypad interaction resulted in an open).
    *   `lid_event` with state `OPENED` (The lid switch sensor detected an open).
*   **Code Reference:** `state/src/iot-streamer/iot-streamer.service.ts`

### 2. Lid Events (Normalized)
The `IotStreamer` processes raw Device Events into normalized `LidEvent` enums. These represent distinct, meaningful physical actions.

*   **Type:** Enum `LidEvent`
*   **Values:**
    *   `LidEvent.LID_OPEN_REQUEST`: A request to open the lid was made (by backend or user).
    *   `LidEvent.LID_OPEN_FAILED`: The lid failed to open after a request (e.g. latch jam, hardware fault).
    *   `LidEvent.LID_OPENED`: The lid has successfully transitioned to the open state.
    *   `LidEvent.LID_CLOSE_REQUEST`: A request to close the lid (rare, usually manual).
*   **Storage:** Stored in `LidCycleEventHistory`.

### 3. Lid Cycle State
A `LidCycle` represents a **session** of interaction with the lid, starting from an intent to open and ending with a successful close (or timeout). It groups multiple `LidEvents` together to provide context (e.g., "This close event belongs to the delivery load that started 1 minute ago").

**Important:** A `LidCycle` is only created when the lid **physically opens** (`LID_OPENED`). "Pre-cycle" events like `LID_OPEN_REQUEST` or `LID_OPEN_FAILED` are recorded in history but are only linked to a cycle *if* the lid eventually opens.

*   **Type:** Enum `LidCycleEvent` (State)
*   **States:**
    *   `LID_CYCLE_INIT`: A cycle has started (e.g., an Open Request or Open Event occurred).
    *   `LID_CYCLE_COMPLETE`: The cycle finished successfully (Lid Closed).
    *   `LID_CYCLE_TIMEOUT`: The cycle was abandoned (Lid left open too long).
*   **Key Field:** `referenceId` (The Delivery ID). This links the physical cycle to a business entity.
*   **Code Reference:** `state/src/lid-cycle/lid-cycle.service.ts`

### 4. Lid Cycle History
This is the audit log (ledger) of all `LidEvents` that occurred within a specific `LidCycle`. It preserves the sequence and timestamps of everything that happened.

*   **Table:** `lid_cycle_event_history`
*   **Relationship:** One `LidCycle` has many `LidCycleEventHistory` entries.
*   **Usage:** Used to reconstruct what happened ("First we requested open, then it failed, then we requested again, then it opened").

## Example: Pinless Loading Flow

| Time | Layer | Event/State | Description |
| :--- | :--- | :--- | :--- |
| T0 | **Device Event** | `pin_entry_event` (lid_opened=true) | Firmware reports touch + open. |
| T0 | **Lid Event** | `LID_OPENED` | Backend normalizes this. |
| T0 | **Lid Cycle** | `LID_CYCLE_INIT` | A new cycle is created. Context: `Source=MAGIC_LID`. |
| T0 | **History** | `LID_OPENED` | Recorded in history linked to the cycle. |
| ... | ... | ... | ... |
| T1 | **Device Event** | `pin_entry_event` (lid_closed) | Firmware reports close. |
| T1 | **Lid Event** | `LID_CLOSED` | Backend normalizes this. |
| T1 | **Lid Cycle** | `LID_CYCLE_COMPLETE` | Cycle is marked complete. |
| T1 | **History** | `LID_CLOSED` | Recorded in history. |

