---
tags:
  - service
  - typescript
  - backend
---
# State Service

*   **Responsibilities:** Tracks the real-time state of [[Robot|robots]] based on events consumed from an SQS queue, which is fed by AWS IoT. It manages the [[Robot]]'s "shadow" (last known state).
*   **AWS Integration:**
    *   **DynamoDB:** Uses tables like `device-state-tracking-dev` to persist [[Robot]] states.
