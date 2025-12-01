---
tags:
  - observability
  - monitoring
  - philosophy
  - strategy
---
# Observability Strategy

A robust observability strategy provides a clear framework for instrumenting our services. It ensures we can effectively monitor system health, alert on user-impacting issues, and efficiently debug problems. This strategy is built on three distinct pillars: **Logs**, **Events**, and **Metrics**.

### Pillar 1: Logs (The Fleeting Record for Debugging)

Logs are verbose, high-cardinality records designed for *investigation* and *debugging* of specific events.

-   **Analogy:** A developer's diary or a plane's flight recorder.
-   **Purpose:** To provide deep, contextual information to answer the question: *"Why did this specific thing happen?"*
-   **System:** Direct to Datadog.
-   **Guiding Principles:**
    1.  **Flexible & Easy:** Developers should feel empowered to add any log with any context they believe will be useful for future debugging. We will **not** enforce a strict schema on the *content* of logs.
    2.  **Structured Format:** All logs **must** be emitted in a structured format (JSON). This makes them parsable and queryable.
    3.  **NEVER Alert on Logs:** We will move away from building alerts on log queries (e.g., `count of logs with "error message"`). This practice is brittle and expensive. Alerts belong to the Metrics tier. The only exception is for extremely rare, critical security or system-failure events.

### Pillar 2: Events (The Business Record for Analytics)

Events are immutable, factual records of significant business milestones. They are designed for *analytics* and long-term trend analysis.

-   **Analogy:** A company's accounting ledger.
-   **Purpose:** To provide a source of truth for business intelligence (BI) and answer the question: *"What has happened in our business over time?"*
-   **System:** RabbitMQ -> S3 -> Redshift (OLAP).
-   **Guiding Principles:**
    1.  **Stable & Schema-Driven:** Events are a contract. They **must** have a well-defined and versioned schema. A change to an event is a breaking change for downstream consumers.
    2.  **Represents a Business Fact:** An event is a "noun" describing something that occurred (e.g., `DeliveryOffered`, `RobotReassigned`, `OrderFulfilled`).
    3.  **Asynchronous by Nature:** This pipeline is not for real-time monitoring. It is for after-the-fact analysis.

### Pillar 3: Metrics (The Real-Time Pulse for Monitoring)

Metrics are numerical aggregations of data over time, designed for real-time *monitoring* and *alerting*.

-   **Analogy:** A car's dashboard (speedometer, temperature gauge).
-   **Purpose:** To provide a high-level view of system health and answer the question: *"Is the system healthy right now?"*
-   **System:** Direct to Datadog via DogStatsD.
-   **Guiding Principles:**
    1.  **Standardized Naming:** Metrics **must** follow a consistent naming convention: `service.component.action_unit` (e.g., `dispatch.planner.reassignments_count`).
    2.  **Low Cardinality Tags:** Tags are for grouping, not for unique identifiers. `reason:offline` is a good tag. `demand_id:<uuid>` is an anti-pattern and must be avoided.
    3.  **The Right Tool for the Job:** Use the four metric types correctly: `Count`, `Gauge`, `Histogram`, and `Distribution`.

### The Unified Strategy: How They Work Together

| Pillar  | Purpose                                | System                          | Structure          | Alerting                                 |
| :------ | :------------------------------------- | :------------------------------ | :----------------- | :--------------------------------------- |
| **Logs**    | **Debugging** & Investigation          | Datadog Logs                    | Flexible (JSON)    | **No** (Except for critical errors)      |
| **Events**  | **Analytics** & Business Intelligence | RabbitMQ -> Redshift (OLAP)     | **Strict Schema**  | **No** (Not real-time)                   |
| **Metrics** | **Monitoring** & Real-time Alerting  | Datadog Metrics                 | **Standardized**   | **Yes** (Based on SLOs and thresholds)   |

**Should we publish all metrics to RabbitMQ?** No. Real-time metrics for Datadog should go directly to Datadog. This path is optimized for low latency. The Event stream via RabbitMQ is optimized for durability and analytics. While an Event in RabbitMQ can *generate* a metric in your data warehouse, the two streams serve different purposes.

**Alignment:** Crucially, when an action generates both an Event and a Metric, they should be aligned. For example, a `RobotReassigned` Event in RabbitMQ should correspond to a `dispatch.planner.reassignments_count` Metric in Datadog. This allows you to correlate real-time alerts with long-term business trends.

### The Alerting Strategy: Based on SLOs

Our alerting philosophy will be based on [[SLO (Service Level Objective)|SLOs]], which are user-centric measures of reliability.

1.  **Define User Journeys:** Identify the critical paths of our service (e.g., "Requesting a Quote," "Confirming a Delivery").
2.  **Create SLOs:** For each journey, define an SLO based on **Metrics**.
    -   *Availability SLO:* "99.9% of quote requests will return a successful (non-5xx) response."
    -   *Latency SLO:* "95% of quote requests will be completed within 500ms."
3.  **Alert on Burn Rate:** Our primary, page-worthy alerts will trigger when our error budget is being consumed too quickly, indicating a real-time, user-impacting issue.
4.  **Lower-Severity Alerts:** We can have lower-priority alerts on system-level metrics (like a spike in reassignments) that notify a Slack channel. These are leading indicators of a problem but are not, in themselves, a breach of our promise to the user.
