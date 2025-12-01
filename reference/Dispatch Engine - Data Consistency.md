---
tags:
  - dispatch-engine
  - data-integrity
  - best-practices
  - prisma
---
# Dispatch Engine - Data Consistency

Ensuring data consistency in the [[Dispatch Engine]] is critical, especially given the distributed nature of the system (involving Postgres, Redis, and external services like [[Operations Service]]).

## Prisma Update Patterns: `null` vs `undefined`

When updating records using Prisma, specifically when dealing with partial updates or "patch" logic, it is crucial to distinguish between `null` and `undefined`.

*   **`null`**: Explicitly sets the database column to `NULL`. Use this when you intend to *erase* data.
*   **`undefined`**: Tells Prisma to **ignore** the field in the update query. The existing value in the database is preserved. Use this for partial updates.

**Risk**: Passing `null` unintentionally (e.g., `params.eta ?? null`) can corrupt data by overwriting valid existing timestamps with `NULL`.
**Best Practice**: Use `params.eta ?? undefined` to ensure that missing parameters result in a no-op for that specific column.

## Trip ID Integrity

A [[Demand]] is typically associated with a [[Trip]] in the [[Operations Service]]. This link is established via the `tripId` field.

### Creation & Lifecycle
*   **Creation**: Ideally, a `tripId` is generated when the [[Demand]] is created (e.g., via `createDeployment` or `createReturnTrip`).
*   **Delivery Demands**: For deliveries, the `tripId` is externally generated and synced.

### Missing Trip IDs
In some edge cases (legacy data, failed trip creation, manual interventions), a [[Demand]] might exist without a `tripId`.
*   **Impact on Activation**: A [[Demand]] generally cannot be *activated* (transition to `Active` status) without a `tripId`, as the `activateTrip` call to the [[Operations Service]] requires it.
*   **Impact on Pending State**: Transitioning to `Pending` (e.g., when a robot is loaded) should ideally *not* be blocked by a missing `tripId` if activation is not immediately requested. Blocking this transition can lead to state mismatches (e.g., [[Dispatch Engine - Orphaned Demands]]).

## Distributed State & Race Conditions

The system manages state across:
1.  **Postgres**: The persistent source of truth for [[Demand]] and [[Robot]] entities.
2.  **Redis**: Used for caching active demands and locking.
3.  **In-Memory**: The `replan` loop operates on a snapshot of state.

### Atomicity
Operations that span multiple logical steps (e.g., [[Dispatch Engine - handleDeliveryLoaded]]) are vulnerable to race conditions or partial failures if not handled atomically or idempotently.
*   **Example**: Assigning a robot (Step 1) and updating demand status (Step 2) must be robust. If Step 1 succeeds but Step 2 fails/aborts, the system enters an inconsistent state.
*   **Mitigation**:
    *   Use database transactions where possible.
    *   Implement state checks (e.g., "Is this demand already Active?") to handle re-entrancy gracefully.
    *   Avoid downgrading state (e.g., `Active` -> `Pending`) unless explicitly intended.

