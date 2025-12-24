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

## Testing & Development

*   **Testing:** See [[State Service Testing]] for commands to run unit and integration tests.
*   **Prisma:** Uses Prisma ORM. See [[Prisma Client Generation in Monorepo]] for generation details.
