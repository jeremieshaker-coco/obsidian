---
tags:
  - metrics
  - catalog
  - dispatch-engine
  - observability
---
# Dispatch Engine Metrics Catalog

This document serves as the central catalog for all key metrics emitted by the [[Dispatch Engine]]. It is the source of truth for metric names, descriptions, and tag usage, ensuring consistency across our [[Observability Strategy]].

### Naming Convention

All metrics **must** follow the `service.component.action_unit` naming convention.
-   **service:** Always `dispatch`
-   **component:** The logical part of the service (e.g., `planner`, `api`)
-   **action_unit:** A noun-verb pair describing what is being measured (e.g., `reassignments_count`, `quote_latency_seconds`)

---

## Planner Metrics

These metrics are emitted from the core replanning loop and provide insights into the health and efficiency of our dispatching logic.

| Metric Name                               | Type         | Description                                                                                             | Tags                 |
| ----------------------------------------- | :----------- | :------------------------------------------------------------------------------------------------------ | :------------------- |
| `dispatch.planner.replan_duration_ms`     | Distribution | Measures the latency of a full `replan` cycle, in milliseconds.                                         | `status` (success/fail) |
| `dispatch.planner.reassignments_count`      | Count        | Incremented every time a [[Demand]] is orphaned and reassigned to a new [[Robot]].                      | `reason`             |
| `dispatch.planner.failures_count`         | Count        | Incremented when the planner fails to generate an estimate for a scheduled [[Demand]].                    | `type`               |
| `dispatch.planner.delay_minutes`          | Gauge        | The delay between a [[Demand]]'s earliest start time and its newly estimated start time (`etd`).          | `type`               |
| `dispatch.planner.orphans_total`          | Gauge        | The total number of orphaned [[Demand|demands]] found at the start of a `replan` cycle.                   | -                    |
| `dispatch.planner.eta_updates_total`      | Gauge        | The total number of ETA updates published for deliveries during a `replan` cycle.                         | -                    |
| `dispatch.planner.activations_total`      | Gauge        | The total number of non-delivery [[Trip|trips]] (pickups, returns) activated during a `replan` cycle.     | -                    |

### Tagging Best Practices

-   **Use Low Cardinality Tags:** Tags should represent a small, fixed set of values (`reason`, `status`, `type`).
-   **AVOID High Cardinality Tags:** Never use unique identifiers like `demand_id` or `robot_id` as tags. This leads to a "metrics explosion" and is an anti-pattern. High-cardinality data belongs in [[Observability Strategy|Logs]].

This catalog should be updated whenever a new metric is added or an existing one is changed to maintain a shared understanding of our observability data.
